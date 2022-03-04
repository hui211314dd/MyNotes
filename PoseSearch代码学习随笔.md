PoseMatching
单个动画已经完成，多个需要Motion Matching配合，目前有bug
MetaData中SamplingRange与AnimState_Block的关系以及Range参数含义等(帧数还是时间？)

```C++
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
```

```C++
// 正如注释里说明的，SequenceStartInterval表示Sequence开头多长时间的动画禁止Transition，这样的话，后面的数据帧会有正确的Past trajectory数据
// SequenceEndInterval表示末尾多长时间的动画禁止Transition, 这样的话，不仅前面的数据帧会有正确的future trajectory,并且可以避免transition后瞬间结束的现象
// 这里有个问题是，PoseMatching中其实不需要这两个参数，而且UPoseSearchSequenceMetaData并没有选项设置，导致SequenceEndInterval一直在使用0.2的默认值，如果这时候我设置SampleRange为[0, 0.2]的话其实是无效的，导致没有帧可用了，准备发个Pull Request修改下；DatabaseAsset可以直接设置

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
```

```C++
// TODO 在FAnimSamplingContext.Init调用后BoneContainer会存储骨骼信息
struct FAnimSamplingContext
{
	// 有限差分法使用的Delta值，用于计算SampleTime - FiniteDelta，SampleTime,SampleTime + FiniteDelta，最后再计算速度信息V(s) ~= (f(s+h) - f(s-h))/2h
	// Time delta used for computing pose derivatives
	static constexpr float FiniteDelta = 1 / 60.0f;

	FBoneContainer BoneContainer;
	
	// TODO 镜像数据使用，以后再补充说明
	TObjectPtr<UMirrorDataTable> MirrorDataTable = nullptr;
	
	// TODO 镜像数据使用，以后再补充说明
	TCustomBoneIndexArray<FCompactPoseBoneIndex, FCompactPoseBoneIndex> CompactPoseMirrorBones;
	
	// TODO 镜像数据使用，以后再补充说明
	TCustomBoneIndexArray<FQuat, FCompactPoseBoneIndex> ComponentSpaceRefRotations;
};
```

```C++
struct FPoseSearchExtrapolationParameters
{
}
```

```C++
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
		// TODO 各个含义？
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

	void Init(const FInput& Input);
	void Process();

    // 下面这些函数在 FSequenceIndexer 会被调用

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
```

```C++
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
```

```C++
// 正如下面注释所提到的，这是个helper函数，用于将time或者distance限定在范围内，返回值FSamplingParam表示出处理后的情况
// This is a helper function used by both time and distance sampling. A schema may specify time or distance
// offsets that are multiple cycles of a clip away from the current pose being sampled.
// And that time or distance offset may before the beginning of the clip (SamplingParam < 0.0f)
// or after the end of the clip (SamplingParam > SamplingParamExtent). So this function
// helps determine how many cycles need to be applied and what the wrapped value should be, clamping
// if necessary.
static FSamplingParam WrapOrClampSamplingParam(bool bCanWrap, float SamplingParamExtent, float SamplingParam)
```

```C++
// 经过Init以及Process函数处理后，Input会赋值为传入的Input，Output会设置为相应的值.
// 该类是BuildIndex计算过程中最核心的类
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
		// TODO Pose Matching中不设置
		const FSequenceSampler* LeadInSequence = nullptr;
		// TODO Pose Matching中不设置
		const FSequenceSampler* FollowUpSequence = nullptr;
		// TODO 是否镜像，Pose Matching中不设置
		bool bMirrored = false;
        // 传入采样的范围，单位为时间而不是帧数,范围clamp在[0, AnimLength]内
		FFloatInterval RequestedSamplingRange = FFloatInterval(0.0f, 0.0f);
		// TODO 正如上面说过，Pose Matching时SequenceEndInterval默认值一直是0.2f,如果采样范围小于0.2f的话，会导致采样失败的...这里要特别注意！
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

	void Init(const FInput& Input);
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
    // TODO 
    FSampleInfo GetSampleInfo(float SampleTime) const;
	FSampleInfo GetSampleInfoRelative(float SampleTime, const FSampleInfo& Origin) const;
	const float GetSampleTimeFromDistance(float Distance) const;
	FTransform MirrorTransform(const FTransform& Transform) const;

    // ----------Process 函数执行流如下---------------
    // 采样开始时会重置FeatureVector
  	void SampleBegin(int32 SampleIdx);

	// 根据Schema设置的SampledBones以及PoseSampleTimes信息，算出SampleIdx的对应的f(s-h), f(s)以及f(s+h)的信息，算出位置和速度，最后设置到FeatureVector
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
```

```C++
// 

/**
* Information about a source animation asset used by a search index.
* Some source animation entries may generate multiple FPoseSearchIndexAsset entries.
**/
USTRUCT()
struct FPoseSearchIndexAsset
{
    // PoseMatching时不设置
    UPROPERTY()
	int32 SourceGroupIdx = INDEX_NONE;

    // 在FPoseSearchIndex::Assets的索引值;
    // PoseMatching时设置为0; TODO 正常PoseSearch时资源在UPoseSearchDatabase的索引？
	// Index of the source asset in search index's container (i.e. UPoseSearchDatabase)
	UPROPERTY()
	int32 SourceAssetIdx = INDEX_NONE;

    // 是否镜像数据
	UPROPERTY()
	bool bMirrored = false;

    // 有效的采样范围值，单位为时间,例如[0, 2.333f]，用于计算Search后QueryResult的TimeOffsetSeconds
	UPROPERTY()
	FFloatInterval SamplingInterval;

    //在FPoseSearchIndex::Values以及FPoseSearchIndex::PoseMetadata的起始Index
	UPROPERTY()
	int32 FirstPoseIdx = INDEX_NONE;

    // 在FPoseSearchIndex::Values以及FPoseSearchIndex::PoseMetadata的Pose的数量
	UPROPERTY()
	int32 NumPoses = 0;
}
```

```C++
// AnimSequence添加的Meta数据，保存时会调用PreSave，之后会调用BuildIndex，通过传入设置好的Schema,SamplingRange以及ExtrapolationParameters生成SearchIndex数据，供Query使用。

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
```

```C++
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
```

```C++
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
    // 如果是PoseSearch(即UPoseSearchDatabase)的话，Assets的数量为多个(TODO 多少个呢？)
	UPROPERTY()
	TArray<FPoseSearchIndexAsset> Assets;
}
```

```C++
// UE::PoseSearch::Search函数的唯一参数，需要填充下查询所需的信息
struct FSearchContext
{
   // Normalized Query values,具体值为FPoseSearchFeatureVectorBuilder.GetNormalizedValues
   TArrayView<const float> QueryValues;
   // TODO
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
   // PoseMatching使用单个AnimSequence时有效，引用动画资源
	const UAnimSequenceBase* SourceSequence = nullptr;
   // PoseSearchDatabase和UPoseSearchSequenceMetaData内部都维护了FPoseSearchIndex的数据，这里直接拿到引用
	const FPoseSearchIndex* SearchIndex = nullptr;
   // TODO
	float MirrorMismatchCost = 0.0f;
}
```

FPoseSearchIndexPreprocessInfo
UPoseSearchSchema

Mirror原理，Preprocess以及Footlock