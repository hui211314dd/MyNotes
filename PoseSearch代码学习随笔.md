```C++
/**
 *  PoseSearch
*/

// 特别重要的类，主要用来处理将features按照feature vector layout格式写入到buffer中，并且会跟踪当前哪些features已经填入了，从而逐步建立feature vector. 还有就是用于在runtime构建search queries
struct FPoseSearchFeatureVectorBuilder
{
   // functions begin
   // ...
   // functions end

   // 数据指向UPoseSearchSchema，Feature处理过程中会反复用到UPoseSearchSchema数据
   UPROPERTY(Transient)
	TWeakObjectPtr<const UPoseSearchSchema> Schema = nullptr;

   // 数组长度为UPoseSearchSchema->Layout.NumFloats
	TArray<float> Values;
   // 数组长度为UPoseSearchSchema->Layout.NumFloats, Values经过Normalized处理后的数据
	TArray<float> ValuesNormalized;
   // Mask标志，那些Features已经录入了，数组长度为数组长度为UPoseSearchSchema->Layout.Features.Num()
	TBitArray<> FeaturesAdded;
   // 已经录入的Features数量
	int32 NumFeaturesAdded = 0;
}

// 正如注释里说明的，SequenceStartInterval表示Sequence开头多长时间的动画禁止Transition，这样的话，后面的数据帧会有正确的Past trajectory数据
// SequenceEndInterval表示末尾多长时间的动画禁止Transition, 这样的话，不仅前面的数据帧会有正确的future trajectory,并且可以避免transition后瞬间结束的现象
// 这里有个问题是，PoseMatching(即UPoseSearchSequenceMetaData)中其实不需要这两个参数，而且UPoseSearchSequenceMetaData并没有选项设置，导致SequenceEndInterval一直在使用0.2的默认值，如果这时候我设置SampleRange为[0, 0.2]的话其实是无效的，导致没有帧可用了，准备发个Pull Request修改下；DatabaseAsset可以直接设置

//  SequenceStartInterval                      SequenceEndInterval
//  *********************++++++++++...+++++++++*******************
USTRUCT()
struct FPoseSearchBlockTransitionParameters
{
	GENERATED_BODY()

	UPROPERTY(EditAnywhere, Category = "Settings")
	float SequenceStartInterval = 0.0f;

	UPROPERTY(EditAnywhere, Category = "Settings")
	float SequenceEndInterval = 0.2f;
};

// 在FAnimSamplingContext.Init调用后BoneContainer会存储骨骼信息，如果配置了MirrorDataTable则初始化其他镜像功能所需的参数
struct FAnimSamplingContext
{
	// 有限差分法使用的Delta值，用于计算SampleTime - FiniteDelta，SampleTime,SampleTime + FiniteDelta，最后再计算速度信息V(s) ~= (f(s+h) - f(s-h))/2h
	// Time delta used for computing pose derivatives
	static constexpr float FiniteDelta = 1 / 60.0f;

	FBoneContainer BoneContainer;
	
	// 镜像功能使用，指向Schema中的MirrorDataTable
	TObjectPtr<UMirrorDataTable> MirrorDataTable = nullptr;
	
	/* 镜像数据使用，记录了BoneContainer中相关骨骼与镜像骨骼的对应表, 比如foot-l对应的CompactPoseBoneIndex为4，对应的镜像骨骼为foot-r,CompactPoseBoneIndex为7
	比如存储如下数据:
	    [0, 0]root->root
	    [1, 1] pelvis->pelvis
	    [2, 5]thigh_l->thigh_r
	    [3, 6]calf_l->calf_r
	    [4, 7]foot_l->foot_r
	    [5, 2]thigh_r->thigh_l
	    [6, 3]calf_r->calf_l
	    [7, 4]foot_r->foot_l
	*/
	TCustomBoneIndexArray<FCompactPoseBoneIndex, FCompactPoseBoneIndex> CompactPoseMirrorBones;
	
	// 镜像数据使用，记录了CompactPoseBoneIndex对应骨骼在绑定姿势下的组件空间旋转信息
	TCustomBoneIndexArray<FQuat, FCompactPoseBoneIndex> ComponentSpaceRefRotations;
};

// 当采样到[0, AnimLength]以外的区域时，通过外推参数预测样点信息，算法首先算出[0, SampleTime]时间内的位移信息，有了位移和旋转信息后，通过SampleTime算出平移速度和旋转角速度，如果平移速度大于等于指定的阈值LinearSpeedThreshold则外推时使用该平移速度，否则速度设置为0; 旋转角速度同理，阈值为AngularSpeedThreshold
USTRUCT()
struct FPoseSearchExtrapolationParameters
{
	GENERATED_BODY()

public:
	// If the angular root motion speed in degrees is below this value, it will be treated as zero.
	UPROPERTY(EditAnywhere, Category = "Settings")
	float AngularSpeedThreshold = 1.0f;
	
	// If the root motion linear speed is below this value, it will be treated as zero.
	UPROPERTY(EditAnywhere, Category = "Settings")
	float LinearSpeedThreshold = 1.0f;

	// Time from sequence start/end used to extrapolate the trajectory.
	UPROPERTY(EditAnywhere, Category = "Settings")
	float SampleTime = 0.05f;
};

// 经过Init以及Process函数处理后，Input会赋值为传入的Input，Output会设置为相应的值.
struct FSequenceSampler
{
public:
	struct FInput
	{
		// 传入上述FAnimSamplingContext的指针
		const FAnimSamplingContext* SamplingContext = nullptr;
		// UPoseSearchSchema的指针
		const UPoseSearchSchema* Schema = nullptr;
		// 动画资源的引用
		const UAnimSequence* Sequence = nullptr;
		// 是否为循环动画
		bool bLoopable = false;
		// 目前是常量60，注意该值和Schema中的SampleRate是相互独立的
		// 区别: DistanceSamplingRate是AccumulatedRootDistance样点的密度，SampleRate是取样的密度，即一个是`种`的密度，一个是`摘`的密度
		int32 DistanceSamplingRate = 60;
		// 外推的控制参数
		FPoseSearchExtrapolationParameters ExtrapolationParameters;
	} Input;

	struct FOutput
	{
		// 每个采样点的累计Root移动距离(注意不是采样点到首个采样点的距离，而是相对于上一个采样点的累计距离)，数值的长度为NumDistanceSamples
		TArray<float> AccumulatedRootDistance;
		// 等于 math.ceil(PlayLength * DistanceSamplingRate) + 1
		int32 NumDistanceSamples = 0;
		// 动画的长度
		float PlayLength = 0.0f;
		// 最终累计移动距离 等于AccumulatedRootDistance.Last()
		float TotalRootDistance = 0.0f;
		// 最后一个采样点与首个采样点的相对Transform
		FTransform TotalRootMotion = FTransform::Identity;
	} Output;

    // 重置成员变量，设置Output中的PlayLength以及NumDistanceSamples
	void Init(const FInput& Input);
	// 调用ProcessRootMotion，该函数主要是遍历动画NumDistanceSamples个采样点，计算累计平移量(AccumulatedRootDistance)，最后计算TotalRootMotion和TotalRootDistance
	void Process();

    // 下面这些函数在 FSequenceIndexer 会被调用

    // 从动画中提取Pose, 值得注意的是这个函数需要处理Mirrored的情况(另外还有一个函数是MirrorTransform)
	// Extracts pose from input sequence and mirrors it if necessary
	void ExtractPose(const FAnimExtractContext& ExtractionCtx, bool bMirrored, FAnimationPoseData& OutAnimPoseData) const;

	// Extracts root transform at the given time, using the extremities of the sequence to extrapolate beyond the 
	// sequence limits when Time is less than zero or greater than the sequence length.
	FTransform ExtractRootTransform(float Time) const;

	// Extracts the accumulated root distance at the given time, using the extremities of the sequence to extrapolate 
	// beyond the sequence limits when Time is less than zero or greater than the sequence length
	float ExtractRootDistance(float Time) const;

	// Extracts notify states inheriting from UAnimNotifyState_PoseSearchBase present in the sequence at Time.
	// The function does not empty NotifyStates before adding new notifies!
	void ExtractPoseSearchNotifyStates(float Time, TArray<class UAnimNotifyState_PoseSearchBase*>& NotifyStates) const;

// private functions
}

struct FSamplingParam
{
    // 经过Wrap以及Clamp处理后的值，可以是时间或者距离
	float WrappedParam = 0.0f;
    // 经过了多少Cycle的处理才落在了合理范围内
	int32 NumCycles = 0;
    // 是否经过Clamp处理
	bool bClamped = false;
	
    // 如果不是循环动画的话，经过clamp处理后的余项存储在这里
	float Extrapolation = 0.0f;
};

// 正如下面注释所提到的，这是个helper函数，用于将time或者distance限定在范围内，返回值FSamplingParam表示出处理后的情况
// This is a helper function used by both time and distance sampling. A schema may specify time or distance
// offsets that are multiple cycles of a clip away from the current pose being sampled.
// And that time or distance offset may before the beginning of the clip (SamplingParam < 0.0f)
// or after the end of the clip (SamplingParam > SamplingParamExtent). So this function
// helps determine how many cycles need to be applied and what the wrapped value should be, clamping
// if necessary.
static FSamplingParam WrapOrClampSamplingParam(bool bCanWrap, float SamplingParamExtent, float SamplingParam)

// 

/**
* FPoseSearchIndexAsset具体指代什么呢？因为大部分情况我们不只是只用Origin单个的资源，可能还有它的Mirror版本，资源本身可能还有弃用的片段比如动捕时的TPose情况，所以资源被拆成了多个Ranges, 而且单个资源可能分配到了多个组里面，比如Idle和Battle组，Asset其实是众多组合可能中的一种情况，比如某个Asset可能就是[0, 20]区间Idle组的Mirror信息，一个UPoseSearchDatatable中FPoseSearchIndexAsset数量大致等于SequencesNum * GroupTags * ValidRangesNum * IsNotMirror)

* Information about a source animation asset used by a search index.
* Some source animation entries may generate multiple FPoseSearchIndexAsset entries.
**/
USTRUCT()
struct FPoseSearchIndexAsset
{
    // 单个AnimSequence PoseMatching(即UPoseSearchSequenceMetaData)时不设置
	// 正常PoseSearch时表示所属组在UPoseSearchDatabase::Groups中的索引值(基于0)
    UPROPERTY()
	int32 SourceGroupIdx = INDEX_NONE;

    // 在FPoseSearchIndex::Assets的索引值;
    // 单个AnimSequence PoseMatching(即UPoseSearchSequenceMetaData)时设置为0; 正常PoseSearch时资源在UPoseSearchDatabase::Sequences的索引值(基于0)
	// Index of the source asset in search index's container (i.e. UPoseSearchDatabase)
	UPROPERTY()
	int32 SourceAssetIdx = INDEX_NONE;

    // 是否镜像数据
	UPROPERTY()
	bool bMirrored = false;

    // 有效的采样范围值，单位为时间,例如[0, 2.333f]，用于计算Search后QueryResult的TimeOffsetSeconds
	UPROPERTY()
	FFloatInterval SamplingInterval;

    //在FPoseSearchIndex::Values以及FPoseSearchIndex::PoseMetadata的起始Index(需要进行一些换算)
	UPROPERTY()
	int32 FirstPoseIdx = INDEX_NONE;

    // 在FPoseSearchIndex::Values以及FPoseSearchIndex::PoseMetadata的Pose的数量
	UPROPERTY()
	int32 NumPoses = 0;
}






// 经过Init以及Process函数处理后，Input会赋值为传入的Input，Output会设置为相应的值.
// 该类是BuildIndex计算过程中最核心的类
// FSequenceIndexer的数量等于FPoseSearchIndexAsset的数量
struct FSequenceIndexer
{
public:
	struct FInput
	{
		// 传入FAnimSamplingContext的指针
		const FAnimSamplingContext* SamplingContext = nullptr;
		// 传入UPoseSearchSchema指针
		const UPoseSearchSchema* Schema = nullptr;
		// 传入主动画的FSequenceSampler指针
		const FSequenceSampler* MainSequence = nullptr;
		// PoseMatching(即UPoseSearchSequenceMetaData)中不设置; 正常PoseSearch时先判断SamplingRange.Min是否为0，如果不为0的话LeadInSequence直接传nullptr即可,否则正常传入设置好的LeadInSequence
		const FSequenceSampler* LeadInSequence = nullptr;
		// PoseMatching(即UPoseSearchSequenceMetaData)中不设置;正常PoseSearch时先判断SamplingRange.Min是否为AnimLength，如果不等的话LeadInSequence直接传nullptr即可,否则正常传入设置好的FollowUpSequence
		const FSequenceSampler* FollowUpSequence = nullptr;
		// 是否镜像数据，PoseMatching(即UPoseSearchSequenceMetaData)中不设置
		bool bMirrored = false;
        // 传入采样的范围，单位为时间而不是帧数,范围clamp在[0, AnimLength]内
		FFloatInterval RequestedSamplingRange = FFloatInterval(0.0f, 0.0f);
		// 正如上面说过，PoseMatching(即UPoseSearchSequenceMetaData)时SequenceEndInterval默认值一直是0.2f,如果采样范围小于0.2f的话，会导致采样失败的...这里要特别注意！
		FPoseSearchBlockTransitionParameters BlockTransitionParameters;
	} Input;

	struct FOutput
	{
		// 根据RequestedSamplingRange提供的采样区间算出首个采样点索引(RequestedSamplingRange.Min * Schema->SampleRate)
		int32 FirstIndexedSample = 0;
		// 根据RequestedSamplingRange提供的采样区间算出最后采样点索引(RequestedSamplingRange.Max * Schema->SampleRate)
		int32 LastIndexedSample = 0;
		// 采样点的数量(LastIndexedSample - FirstIndexedSample + 1)
		int32 NumIndexedPoses = 0;
		// 存储每个采样点的FeatureVector情况, 数组长度等于Schema->Layout.NumFloats * NumIndexedPoses
		TArray<float> FeatureVectorTable;
		// 存储每个采样点的PoseMetadata情况，数组长度等于NumIndexedPoses
		TArray<FPoseSearchPoseMetadata> PoseMetadata;
	} Output;

    // 重置成员变量，设置Output中的FirstIndexedSample, LastIndexedSample以及NumIndexedPoses
	void Init(const FInput& Input);
	// 对于每次采样点依次调用SampleBegin, AddPoseFeatures, AddTrajectoryTimeFeatures, AddTrajectoryDistanceFeatures, AddMetadata和SampleEnd
	void Process();

private:
    // Process处理过程中的中间变量，用于存储数据，最后拷贝到FeatureVectorTable和PoseMetadata中
	FPoseSearchFeatureVectorBuilder FeatureVector;
	FPoseSearchPoseMetadata Metadata;

	struct FSampleInfo
	{
		const FSequenceSampler* Clip = nullptr;
		FTransform RootTransform;
		float ClipTime = 0.0f;
		float RootDistance = 0.0f;

		bool IsValid() const { return Clip != nullptr; }
	};

    // private functions
    // ----------Helper functions--------------------
    /* 获取SampleTime采样点的信息
	   Input:
	        SampleTime: 任意正负值
	   Return:
	        FSampleInfo::Clip          SampleTime对应的Clip，可能为LeadInSequence，MainSequence或者FollowUpSequence
			FSampleInfo::RootTransform SampleTime相对于0采样点的Root偏移
			FSampleInfo::ClipTime      SampleTime对应到Clip的哪个时间点上
			FSampleInfo::RootDistance  SampleTime相对于0采样点的Distance偏移
	 */
    FSampleInfo GetSampleInfo(float SampleTime) const;

    /*
	    获取SampleTime的采样点信息，然后将其中的RootTransform以及RootDistance设置为相对于Origin的位置
		TODO RootDistance的计算是不是反了？
	*/
	FSampleInfo GetSampleInfoRelative(float SampleTime, const FSampleInfo& Origin) const;
	const float GetSampleTimeFromDistance(float Distance) const;
	FTransform MirrorTransform(const FTransform& Transform) const;

    // ----------Process 函数执行流如下---------------
    // 采样开始时会重置FeatureVector
  	void SampleBegin(int32 SampleIdx);

	// 根据Schema设置的SampledBones以及PoseSampleTimes信息，算出SampleIdx的对应的f(s-h), f(s)以及f(s+h)的信息，算出位置和速度，最后设置到FeatureVector
	// 需要注意的是，这个过程中计算出来的位置信息都是相对于当前采样点，同样速度方向也是，这样做是为了将来Feature比较时方便
	void AddPoseFeatures(int32 SampleIdx);

	// TODO
	void AddTrajectoryTimeFeatures(int32 SampleIdx);
	void AddTrajectoryDistanceFeatures(int32 SampleIdx);

    // 算出SampleIdx对应采样点的FPoseSearchPoseMetadata, 储存到内部变量MetaData中
    // 计算过程在FSequenceIndexer::AddMetadata函数内. 如果动画为非循环动画，那么RequestedSamplingRange参数会参与到首尾是否BlockTransition的计算(循环的话RequestedSamplingRange不生效)，接下来检查该采样点有哪些NotifyState,比如UAnimNotifyState_PoseSearchBlockTransition的话，说明BlockTransition,如果是UAnimNotifyState_PoseSearchModifyCost的话，说明有额外消耗.(可以看到NotifyState两种用法，一是接收NotfyBegin/Tick/End响应事件，二是仅仅作为标记，外部逻辑通过TriggerTime以及EndTriggerTime自行判断使用)
	void AddMetadata(int32 SampleIdx);

    // 将FeatureVector和MetaData存储到SampleIdx对应的FeatureVectorTable和PoseMetadata中
	void SampleEnd(int32 SampleIdx);
	// ---------Process 执行结束----------------------
    //  ...
}

// AnimSequence添加的Meta数据，保存时会调用PreSave，之后会调用BuildIndex，根据设置好的Schema,SamplingRange以及ExtrapolationParameters生成SearchIndex数据，供Query使用。
// 所以说UPoseSearchSequenceMetaData最核心的成员其实是SearchIndex
UCLASS(BlueprintType, Category = "Animation|Pose Search", Experimental)
class POSESEARCH_API UPoseSearchSequenceMetaData : public UAnimMetaData
{
	GENERATED_BODY()
public:

	UPROPERTY(EditAnywhere, Category="Settings")
	TObjectPtr<const UPoseSearchSchema> Schema = nullptr;

	UPROPERTY(EditAnywhere, Category="Settings")
	FFloatInterval SamplingRange = FFloatInterval(0.0f, 0.0f);

	UPROPERTY(EditAnywhere, Category = "Settings")
	FPoseSearchExtrapolationParameters ExtrapolationParameters;

	UPROPERTY()
	FPoseSearchIndex SearchIndex;

	// functions begin
    // ...

public: // UObject
	virtual void PreSave(FObjectPreSaveContext ObjectSaveContext) override;
   
   // functions end
};





struct FPoseSearchWeightParams
{}




// 返回DbSequence.Sequence中的有效采样范围，如果动画中有UAnimNotifyState_PoseSearchExcludeFromDatabase，则分成多个Range返回
// 距离动画SamplingRange设置的是[0, 100],同时UAnimNotifyState_PoseSearchExcludeFromDatabase设置区域为[20, 50],那么返回值为[0, 20)和(50, 100]
void FindValidSequenceIntervals(const FPoseSearchDatabaseSequence& DbSequence, TArray<FFloatRange>& ValidRanges)




// 当保存UPoseSearchDatabase资源时会调用PreSave函数，根据设置项生成SearchIndex数据，供Query使用。
// 正如UPoseSearchSequenceMetaData里提到的，UPoseSearchDatabase的核心成员也是SearchIndex
UCLASS(BlueprintType, Category = "Animation|Pose Search", Experimental)
class POSESEARCH_API UPoseSearchDatabase : public UDataAsset
{
	GENERATED_BODY()
public:

	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category="Database")
	const UPoseSearchSchema* Schema;

	UPROPERTY(EditAnywhere, Category = "Database")
	FPoseSearchWeightParams DefaultWeights;

    // 可以参考FSearchContext::MirrorMismatchCost的解释
	// If there's a mirroring mismatch between the currently playing sequence and a search candidate, this cost will be 
	// added to the candidate, making it less likely to be selected
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Database")
	float MirroringMismatchCost = 0.0f;

	UPROPERTY(EditAnywhere, Category = "Database")
	FPoseSearchExtrapolationParameters ExtrapolationParameters;

	UPROPERTY(EditAnywhere, Category = "Database")
	FPoseSearchBlockTransitionParameters BlockTransitionParameters;

	UPROPERTY(EditAnywhere, Category = "Database")
	TArray<FPoseSearchDatabaseGroup> Groups;

	// Drag and drop animations here to add them in bulk to Sequences
	UPROPERTY(EditAnywhere, Category = "Database", DisplayName="Drag And Drop Anims Here")
	TArray<TObjectPtr<UAnimSequence>> SimpleSequences;

	UPROPERTY(EditAnywhere, Category="Database")
	TArray<FPoseSearchDatabaseSequence> Sequences;

	UPROPERTY()
	FPoseSearchIndex SearchIndex;

	int32 FindSequenceForPose(int32 PoseIdx) const;
	float GetSequenceLength(int32 DbSequenceIdx) const;
	bool DoesSequenceLoop(int32 DbSequenceIdx) const;

	bool IsValidForIndexing() const;
	bool IsValidForSearch() const;

	int32 GetPoseIndexFromAssetTime(float AssetTime, const FPoseSearchIndexAsset* SearchIndexAsset) const;
	float GetTimeOffset(int32 PoseIdx, const FPoseSearchIndexAsset* SearchIndexAsset = nullptr) const;
	const FPoseSearchDatabaseSequence& GetSourceAsset(const FPoseSearchIndexAsset* SearchIndexAsset) const;

public: // UObject
    // 会调用BuildIndex(UPoseSearchDatabase* Database)构建SearchIndex数据
	virtual void PreSave(FObjectPreSaveContext ObjectSaveContext) override;

#if WITH_EDITOR
	virtual void PostEditChangeProperty(struct FPropertyChangedEvent& PropertyChangedEvent) override;
#endif

private:
	void CollectSimpleSequences();

public:
    // 通过evaluating Sequences Array的数据来填充FPoseSearchIndex::Assets数据
	/*
	   细节如下:
	        遍历Sequences中的每个动画资源（主要是FPoseSearchDatabaseSequence::Sequence,LeadInSequence和FollowUpSequence在此并没有做处理）
			     1. 判断是需要原始数据还是镜像数据还是两者都要
				 2. 存储该Sequence存在哪些GroupTags（一个Sequence可能有多个Tag，比如一个Battle_Idle动画可能有BattleTag和IdleTag）存储这些Tag在UPoseSearchDatabase::Groups的索引值，需要注意的是如果Sequence打的Tag没有在Groups中找到，那么会使用默认的Weights，并且在保存时给与警告
				 3. 到这一步时，我们已经知道了这个Sequence 是否需要镜像数据并且归于哪些Group的信息
				 4. 调用FindValidSequenceIntervals获取该Sequence的ValidRanges(TODO 调用可以提到外边)
				 5. 遍历GroupIndices，ValidRanges以及是否镜像，向FPoseSearchIndex::Assets添加项

			没有配置MirrorDataTable的错误处理以及Sequence打的Tag没有在Groups中找到的警告逻辑等
	*/
	bool TryInitSearchIndexAssets();
};

// AnimSequence中每个采样点都有对应的FPoseSearchPoseMetadata用来影响Query结果，比如Flags如果是BlockTransition的话，说明这个采样点不会被作为结果返回；CostAddend表示这一采样点有额外的Cost(负值表示这个采样点更容易被选中，正值表示这个采样点Cost消耗更大，不那么被人喜欢)
USTRUCT()
struct POSESEARCH_API FPoseSearchPoseMetadata
{
	GENERATED_BODY()

	UPROPERTY()
	EPoseSearchPoseFlags Flags = EPoseSearchPoseFlags::None;

	UPROPERTY()
	float CostAddend = 0.0f;
};

// 核心类之一
struct FPoseSearchIndex
{
   // functions begin
   // ...
   // functions end

   // 总的采样或者Pose数量
   // 如果是PoseMatching(即UPoseSearchSequenceMetaData)的话，为一个动画资源Pose数量
   // 如果是PoseSearch(即UPoseSearchDatabase)的话，为UPoseSearchDatabase所有动画资源Pose的数量和
   UPROPERTY()
	int32 NumPoses = 0;

    // 总的FeatureVector集合
    // 如果是PoseMatching(即UPoseSearchSequenceMetaData)的话，为一个动画资源中的FeatureVectorTable
    // 如果是PoseSearch(即UPoseSearchDatabase)的话，为UPoseSearchDatabase所有动画资源FeatureVectorTable集合
	UPROPERTY()
	TArray<float> Values;

    // 总的PoseMetaData集合
    // 如果是PoseMatching(即UPoseSearchSequenceMetaData)的话，为一个动画资源中的PoseMetadata
    // 如果是PoseSearch(即UPoseSearchDatabase)的话，为UPoseSearchDatabase所有动画资源PoseMetadata集合
	UPROPERTY()
	TArray<FPoseSearchPoseMetadata> PoseMetadata;

	UPROPERTY()
	TObjectPtr<const UPoseSearchSchema> Schema = nullptr;

    // TODO 
	UPROPERTY()
	FPoseSearchIndexPreprocessInfo PreprocessInfo;

    // 每个动画资源对应的信息,这些信息供SearchIndex使用
    // 如果是PoseMatching(即UPoseSearchSequenceMetaData)的话，Assets仅有一项
    // 如果是PoseSearch(即UPoseSearchDatabase)的话，Assets的数量为多个(可以参考TryInitSearchIndexAssets的注释, 数量大致等于SequencesNum * GroupTags * ValidRangesNum * IsNotMirror)
	UPROPERTY()
	TArray<FPoseSearchIndexAsset> Assets;
}

// UE::PoseSearch::Search函数的唯一参数，需要填充下查询所需的信息
struct FSearchContext
{
   // Normalized Query values,具体值为FPoseSearchFeatureVectorBuilder.GetNormalizedValues
   TArrayView<const float> QueryValues;

   /* 查询镜像的请求
       如果请求为Indifferent表示是不是镜像无所谓，这时候候选Pose没有MirrorMismatchCost消耗
	   如果请求为TrueValue表示希望候选Pose是镜像数据，如果这时候候选Pose不是镜像数据而是源数据，需要增加MirrorMismatchCost额外消耗
	   如果请求为FalseValue表示希望候选Pose是源数据，如果这时候候选Pose不是源数据而是镜像数据，需要增加MirrorMismatchCost额外消耗
	*/
	EPoseSearchBooleanRequest QueryMirrorRequest = EPoseSearchBooleanRequest::Indifferent;
	// TODO
   const FPoseSearchWeightsContext* WeightsContext = nullptr;
   // TODO
	const FGameplayTagQuery* DatabaseTagQuery = nullptr;
   // TODO
	FDebugDrawParams DebugDrawParams;

// functions begin
// ...
// functions end
private:
   // MotionMatching使用PoseSearchDatabase时有效，引用db数据
	const UPoseSearchDatabase* SourceDatabase = nullptr;

   // PoseMatching(即UPoseSearchSequenceMetaData)时有效，引用动画资源
	const UAnimSequenceBase* SourceSequence = nullptr;

   // PoseSearchDatabase和UPoseSearchSequenceMetaData内部都维护了FPoseSearchIndex的数据，这里直接拿到引用
	const FPoseSearchIndex* SearchIndex = nullptr;

   // 当QueryMirrorRequest请求与候选Pose不匹配时的额外消耗，外部无法设置
   // PoseMatching(即UPoseSearchSequenceMetaData)时为0
   // PoseSearch时一般赋值为UPoseSearchDatabase中的MirroringMismatchCost
	float MirrorMismatchCost = 0.0f;
}

// TODO
struct FPoseCost
{
	float Dissimilarity = MAX_flt;
	float CostAddend = 0.0f;
	float TotalCost = MAX_flt;
	bool operator<(const FPoseCost& Other) const { return TotalCost < Other.TotalCost; }
};

// TODO
struct FSearchResult
{
	FPoseCost PoseCost;
	int32 PoseIdx = INDEX_NONE;
	const FPoseSearchIndexAsset* SearchIndexAsset = nullptr;
	float TimeOffsetSeconds = 0.0f;

	bool IsValid() const { return PoseIdx >= 0; }
}

// 拥有UPoseSearchSequenceMetaData的AnimSequence保存时调用，用于构建UPoseSearchSequenceMetaData的核心成员SearchIndex
bool BuildIndex(const UAnimSequence* Sequence, UPoseSearchSequenceMetaData* SequenceMetaData)

// UPoseSearchDatabase PreSave时调用，用于构建UPoseSearchDatabase的核心成员SearchIndex
bool BuildIndex(UPoseSearchDatabase* Database)

// PoseSearch核心函数，SearchContext为查询所需的参数，FSearchResult结果返回最接近查询要求的Pose信息
FSearchResult Search(FSearchContext& SearchContext)
//---------------------------------------------------------------------------------------------------------------


/**
 *  AnimNode_MotionMatching
*/


//---------------------------------------------------------------------------------------------------------------


/**
 *  PoseSearchLibrary
*/

// 
struct FMotionMatchingPoseStepper
{
	FSearchResult Result;
	bool bJumpRequired = false;

	bool CanContinue() const
	{
		return Result.IsValid();
	}

	void Reset()
	{
		Result = UE::PoseSearch::FSearchResult();
		bJumpRequired = false;
	}

    // TODO LIHUI Result.TimeOffsetSeconds = State.AssetPlayerTime是否赋值错误？
	void Update(const FAnimationUpdateContext& UpdateContext, const struct FMotionMatchingState& State);
};

// 
USTRUCT(BlueprintType, Category = "Animation|Pose Search")
struct POSESEARCH_API FMotionMatchingSettings
{
	GENERATED_BODY()

	// Dynamic weights for influencing pose selection
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=Settings, meta=(PinHiddenByDefault))
	FPoseSearchDynamicWeightParams Weights;

    // 当需要blend out到某个新姿势时，这里控制的是融合的时间，因为使用的是惯性化插值所以MotionMatching Node后面需要跟上一个Inertialization Node
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=Settings, meta=(ClampMin="0"))
	float BlendTime = 0.2f;

    // 如果说当前播放的Pose与Search后TargetPose在Mirror属性上不一致，并且MirrorChangeBlendTime大于0，那么不再使用上面的BlendTime而是使用MirrorChangeBlendTime
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = Settings, meta = (ClampMin = "0", DislayAfter = "BlendTime"))
	float MirrorChangeBlendTime = 0.0f;
	
	// Don't jump to poses that are less than this many seconds away
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=Settings, meta=(ClampMin="0"))
	float PoseJumpThresholdTime = 1.f;

	// Minimum amount of time to wait between pose search queries
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=Settings, meta=(ClampMin="0"))
	float SearchThrottleTime = 0.1f;

	// How much better the search result must be compared to the current pose in order to jump to it
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=Settings, meta=(ClampMin="0", ClampMax="100"))
	float MinPercentImprovement = 40.0f;
};

// 
USTRUCT(BlueprintType, Category="Animation|Pose Search")
struct POSESEARCH_API FMotionMatchingState
{
	GENERATED_BODY()

	// Initializes the minimum required motion matching state
	bool InitNewDatabaseSearch(const UPoseSearchDatabase* Database, float SearchThrottleTime, FText* OutError);

	// Adds trajectory prediction and history information to ComposedQuery
	void ComposeQuery(const UPoseSearchDatabase* Database, const FTrajectorySampleRange& Trajectory);

	// Internally stores the 'jump' to a new pose/sequence index and asset time for evaluation
	void JumpToPose(const FAnimationUpdateContext& Context, const FMotionMatchingSettings& Settings, const UE::PoseSearch::FSearchResult& Result);

	const FPoseSearchIndexAsset* GetCurrentSearchIndexAsset() const;

	float ComputeJumpBlendTime(const UE::PoseSearch::FSearchResult& Result, const FMotionMatchingSettings& Settings) const;

	// The current pose we're playing from the database
	UPROPERTY(Transient)
	int32 DbPoseIdx = INDEX_NONE;

	// The current animation we're playing from the database
	UPROPERTY(Transient)
	int32 SearchIndexAssetIdx = INDEX_NONE;

	// The current query feature vector used to search the database for pose candidates
	UPROPERTY(Transient)
	FPoseSearchFeatureVectorBuilder ComposedQuery;

	// Precomputed runtime weights
	UPROPERTY(Transient)
	FPoseSearchWeightsContext WeightsContext;

	// When the database changes, the search parameters are reset
	UPROPERTY(Transient)
	TWeakObjectPtr<const UPoseSearchDatabase> CurrentDatabase = nullptr;

	// Time since the last pose jump
	UPROPERTY(Transient)
	float ElapsedPoseJumpTime = 0.f;

	// Current time within the asset player node
	UPROPERTY(Transient)
	float AssetPlayerTime = 0.f;

	// Evaluation flags relevant to the state of motion matching
	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category=State)
	EMotionMatchingFlags Flags = EMotionMatchingFlags::None;
};
//---------------------------------------------------------------------------------------------------------------
```




FPoseSearchIndexPreprocessInfo
UPoseSearchSchema

FDynamicPlayRateSettings
FMotionMatchingSettings

PoseMatching文档
单个动画已经完成，多个需要Motion Matching配合，目前有bug
MetaData中SamplingRange与AnimState_Block的关系以及Range参数含义等(帧数还是时间？)

MultiPoseMatching配合AnimState_BlockTransition如何使用？(如何设置仅仅查询一次，然后顺序播放即可)
Group如何使用？
Preprocess，Distance的理解以及应用

Mirror原理(重点并且细致的剖析)，资料有MMDemo, ControlRig/Maya生成Mirror逻辑，PoseSearch等, MirrorTransform？, LU停步动画Mirrored后有位移，bug？
Footlock
修复MotionMatching Node填入DB后不生效的bug