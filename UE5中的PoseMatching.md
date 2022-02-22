UE5中MotionMatching(三) Pose Matching的使用

## 前言
本篇主要讲述的是UE5中如何使用Pose Matching，并不会讲代码的实现细节，后面的系列文章会分享实现细节。阅读本文前建议先阅读[游戏开发中的Pose Matching](https://zhuanlan.zhihu.com/p/424382326)从而了解Pose Matching的原理和应用。

我的UE5-Main是在2022.2.12日更新，而且PoseSearch(UE5对MotionMaching的称呼)本身就处于试验阶段，所以不保证将来是否会有大的改动。

PoseSearch插件路径：UnrealEngine\Engine\Plugins\Experimental\Animation\PoseSearch

如果你对Motion Matching感兴趣，可以看下我的其他文章。

[Motion Matching 中的代码驱动移动和动画驱动移动](https://zhuanlan.zhihu.com/p/432663486)

[《荣耀战魂》中的Motion Matching](https://zhuanlan.zhihu.com/p/401890149)

[《最后生还者2》中的Motion Matching](https://zhuanlan.zhihu.com/p/403923793)

[《Control》中的Motion Matching](https://zhuanlan.zhihu.com/p/405873194)

[游戏开发中的Pose Matching](https://zhuanlan.zhihu.com/p/424382326)

[MotionMatching中的DataNormalization](https://zhuanlan.zhihu.com/p/414438466)

[UE5中MotionMatching(一) MotionTrajectory](https://zhuanlan.zhihu.com/p/453659782)

[UE5中MotionMatching(二) 创建可运行的PoseSearch工程](https://zhuanlan.zhihu.com/p/455983339)

## 正文
### 数据源为单个动画文件
我们仍然以[游戏开发中的Pose Matching](https://zhuanlan.zhihu.com/p/424382326)中的停步举例，我们先假定停步时左脚在空中，那么会使用名为RunFwdStop_LU(LeftFootUp)的动画作为过渡动画

{视频1 RunFwdStop_LU}

虽然我们假定过渡前左脚在空中，但不知道左脚在空中的哪个位置，如果RunFwdStop_LU始终从第一帧开始播放的话，有可能会出现过渡不协调的问题，从视频的慢回放中可以看到在动画切换时腿部有回拉的表现

{视频2 正常Blend带来的问题}

Pose Matching可以帮助我们解决这个问题，它会帮助我们找到最合适的动画起始帧开始播放，效果如下:

{视频3 单个动画资源的Pose Matching}

慢回放可以看到，腿部不再回拉了

{视频4 正常Pose Matching的慢回放}

那么UE5中应该如何配置呢？如果数据源为了单个动画的话，配置起来较为简单
1. 启用PoseSearch插件后，创建Data Asset选择PoseSearchSchema,假设命名为PoseMatchingSchema,根据需求设置好骨骼信息, Trajectory相关的无需设置
   ![](./UE5PoseMatchingPic/1.png)
   
   ![](./UE5PoseMatchingPic/2.png)

2. 打开RunFwdStop_LU动画资源，
### 数据源为多个动画文件


## 注意事项
特别早期，各种bug
MultiPoseMatching
修改Schema或者DB后需要全部保存下，否则Motion Matching会出现TPose情况

## 下一步

如果文章有错误，记得联系我~







## 临时记录
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
// 这里有个问题是，PoseMatching中其实不需要这两个参数，而且UPoseSearchSequenceMetaData并没有选项设置，导致SequenceEndInterval一直在使用0.2的默认值，如果这时候我设置SampleRange为[0, 0.2]的话其实是无效的，导致没有帧可用了，准备发个Pull Request修改下；Database可以直接设置

//  SequenceStartInterval                      SequenceEndInterval
//  *********************++++++++++...+++++++++*******************
USTRUCT()
struct FPoseSearchBlockTransitionParameters
{
	GENERATED_BODY()

	// Excluding the beginning of sequences can help ensure an exact past trajectory is used when building the features
	UPROPERTY(EditAnywhere, Category = "Settings")
	float SequenceStartInterval = 0.0f;

	// Excluding the end of sequences help ensure an exact future trajectory, and also prevents the selection of
	// a sequence which will end too soon to be worth selecting.
	UPROPERTY(EditAnywhere, Category = "Settings")
	float SequenceEndInterval = 0.2f;
};
```

```C++
struct FAnimSamplingContext
{
	// Time delta used for computing pose derivatives
	static constexpr float FiniteDelta = 1 / 60.0f;

	FBoneContainer BoneContainer;
	
	// Mirror data table pointer copied from Schema for convenience
	TObjectPtr<UMirrorDataTable> MirrorDataTable = nullptr;
	
	// Compact pose format of Mirror Bone Map
	TCustomBoneIndexArray<FCompactPoseBoneIndex, FCompactPoseBoneIndex> CompactPoseMirrorBones;
	
	// Pre-calculated component space rotations of reference pose, which allows mirror to work with any joint orientation
	// Only initialized and used when a mirroring table is specified
	TCustomBoneIndexArray<FQuat, FCompactPoseBoneIndex> ComponentSpaceRefRotations;
	
	void Init(const UPoseSearchSchema* Schema);
	
	FTransform MirrorTransform(const FTransform& Transform) const;

private:
	void FillCompactPoseAndComponentRefRotations();
};
```

```C++
struct FPoseSearchExtrapolationParameters
{
}
```

```C++
struct FSequenceSampler
{}
```

```C++
struct FSequenceIndexer
{}
```

```C++
struct FPoseSearchIndexAsset
{}
```

```C++
// AnimSequence添加的Meta数据，保存时会调用PreSave，之后会调用BuildIndex，通过传入设置好的Schema,SamplingRange以及ExtrapolationParameters生成SearchIndex数据，供Query使用
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
// 核心类之一
struct FPoseSearchIndex
{
   // functions begin
   // ...
   // functions end

   UPROPERTY()
	int32 NumPoses = 0;

	UPROPERTY()
	TArray<float> Values;

	UPROPERTY()
	TArray<FPoseSearchPoseMetadata> PoseMetadata;

	UPROPERTY()
	TObjectPtr<const UPoseSearchSchema> Schema = nullptr;

	UPROPERTY()
	FPoseSearchIndexPreprocessInfo PreprocessInfo;

	UPROPERTY()
	TArray<FPoseSearchIndexAsset> Assets;
}
```

```C++
// Search函数的唯一参数，需要填充下查询所需的信息
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
FPoseSearchPoseMetadata
UPoseSearchSchema
