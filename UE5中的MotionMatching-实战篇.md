# UE5中的MotionMatching-实践(一)

## 前言
之前写的大都是MotionMatching各个模块如何使用，如何实现，如何配置参数等文档，实践太少。真正实践后，才能切身感受到MotionMatching的优点(当然还有缺点)，才能碰到众多想象不到的问题。要知道，**MotionMatching从能跑到商用有很长很长的路要走**，而且这段路上的坑极其多，很多公司也是走到半路就放弃了。我写这个系列就是提前踩踩坑，踩坑的过程记录下来，方便后来人能够坚持走下去。

我的UE5-Main是在2022.8.22日更新的，PoseSearch(UE5对MotionMaching的称呼)本身就处于试验阶段，所以不确定将来是否会有大的改动(其实最近一段时间一直有提交)。

PoseSearch插件路径：UnrealEngine\Engine\Plugins\Experimental\Animation\PoseSearch

如果你对MotionMatching感兴趣，可以看下我的其他文章。

[MotionMatching 中的代码驱动移动和动画驱动移动](https://zhuanlan.zhihu.com/p/432663486)

[《荣耀战魂》中的MotionMatching](https://zhuanlan.zhihu.com/p/401890149)

[《最后生还者2》中的MotionMatching](https://zhuanlan.zhihu.com/p/403923793)

[《Control》中的MotionMatching](https://zhuanlan.zhihu.com/p/405873194)

[游戏开发中的Pose Matching](https://zhuanlan.zhihu.com/p/424382326)

[MotionMatching中的DataNormalization](https://zhuanlan.zhihu.com/p/414438466)

[UE5中的MotionMatching(一) MotionTrajectory](https://zhuanlan.zhihu.com/p/453659782)

[UE5中的MotionMatching(二) 创建可运行的PoseSearch工程](https://zhuanlan.zhihu.com/p/455983339)

[UE5中的MotionMatching(三) PoseMatching](https://zhuanlan.zhihu.com/p/492266731)

[UE5中的MotionMatching(四) MotionMatching](https://zhuanlan.zhihu.com/p/507268359)

[UE5中的MotionMatching(五) Mirroring Animation](https://zhuanlan.zhihu.com/p/516358714)

[UE5中的MotionMatching(六) RewindDebugger](https://zhuanlan.zhihu.com/p/549866674)

## 新建测试关卡
测试关卡在我们实践中特别有用，它就像测试人员使用checklist一样，我们要做的就是保证每次调整完参数或者导入新的动画库后原有功能依然正常运行，不能出现按下葫芦浮起瓢的情况。
![TestLevel](.\UE5MotionMatchingPracticePic/TestLevel.png)

还有就是为了方便测试各种情景，会在测试关卡中摆放许多障碍物或者目标路线等,以《最后生还者2》举例：

![TLOU2TestLevel](.\UE5MotionMatchingPracticePic/TLOU2TestLevel.png)

{TLOU2TestLevel.mp4}

## DebugPanel
有的时候希望能在游戏中对一些参数进行微调，而不是退出游戏-调整参数-启动游戏的流程，这样迭代会更快一些，所以有必要创建一个DebugPanel始终显示在最上方，方便调试。

![DebugPanel](.\UE5MotionMatchingPracticePic/DebugPanel.png)

## 动画资源
我们先从简单的动画入手，从虚幻商城中找到了这个动画库[MovementAnisetPro](https://www.unrealengine.com/marketplace/zh-CN/product/movement-animset-pro)并且我们仅仅使用它们Walk相关的几个动画并进行本次的测试。

## 工程的基本设置
Schema中Trajectory采样点为5个，时间分别为-0.25, 0, 0.15, 0.35, 0.45, Flags包括Velocity, Position以及FacingDirection，Pose的话骨骼设置为foot_l和foot_r, Flags包括Velocity以及Position, 采样时间仅仅设置0即可;

## 稳定的Idle
向Database中导入Idle动画

我们碰到的第一个需求就是没有任何输入的情况下，MotionMatching应该循环播放Idle动画

![Idle](.\UE5MotionMatchingPracticePic/Idle.png)

如果你是8月20号之后更新的代码，这里已经没问题了，因为我在之前向官方反馈了比较Cost时数值不稳定的问题，开发者[Braeden Shosa](https://twitter.com/BraedenShosa)随后[解决了这个问题](https://github.com/EpicGames/UnrealEngine/commit/55882c902207057084f0e328427866d602c0465f)，所以这里就不用再谈了。

## 起步并直线行走

向Database中导入WalkFwdStart以及WalkFwdLoop

![WalkFwdStart](.\UE5MotionMatchingPracticePic/WalkFwdStart.png)

刚开始的时候导入的并不是WalkFwdLoop动画而是一个动捕的Walk动画，并且该动画前后几帧都有TPose, 我们希望在设置SampleRange时截断他们，并且Range.Min和Range.Max帧相同，当时希望播放到最后帧Range.Max后紧接着播放Range.Min，为了支持这个功能，我们特意将该动画的Loop设置为true，但是事与愿违，运行时发现播放帧的顺序为1，2，3，4，5，6，5，6，5，6.....播放到末尾第6帧后进行了查询，发现第5帧为理想跳转帧，后面5，6一直循环，我们并不希望这样，因为我们希望从第1帧开始播放，通过代码(FMotionMatchingState::CanAdvance)我们发现SampleRange截断后再Advance为false，这样会紧接着调整到Search的结果上,因此会造成上面的5,6循环问题。所以解决方案是我们导入引擎前就要保证这个动画是循环动画并且首尾衔接，这样就不要设置SampleRange。如何理解呢？AnimSequence中的Loop表示这个动画本身为首尾衔接的Loop动画，导入到Database中设置了SampleRange，这时候不能指望Range内的动画依然可以首尾衔接。所以结论就是：**如果想给Database中导入一个循环动画，那么该动画在DDC制作时最好已经是成品了**

{稳定输入下WalkLoop循环播放.mp4}

## WalkFwdLoop生成Mirrored版本数据

WalkFwdLoop MirrorOption设置为OriginalAndMirrored

某些功能需要我们生成镜像版本的数据，我们再次运行游戏，稳定输入下角色走路一直在JumpPose导致脚部有滑步

{WalkLoop启用Mirrored后发现即使稳定输入下也在反复切换Pose.mp4}

我们通过RewindDebugger发现切换的原因是MirroredWalkFwdLoop的某一个PoseCost值为0.001，而ContinuityPoseCost为0.003，所以系统认定MirroredWalkFwdLoop的Pose比ContinuityPose更加适合，所以选择了跳转。如此反复导致了Pose的频繁切换，如何解决呢？一种可行的解决方法是增加MirroringMismatchCost,这样如果Origin动画和Mirrored动画相互切换时就产生了额外的Cost,我们设置为0.02即可。设置后我们发现稳定输入的情况下，不再频繁切换Pose了

## 站立状态下向不同方向起步

通过WalkFwdStart, WalkFwdStart90_L, WalkFwdStart135_L, WalkFwdStart180_L创建BlendSpace1D，命名为BSWalkFwdLeftStart，引入到Database中并将MirrorOption设置为OriginalAndMirrored, UseGridForSampling设置为false, NumberOfHorizontalSamples设置为20, NumberOfVerticalSamples设置为1

上面我们仅仅导入了WalkFwdStart动画处理向前起步的问题，但游戏中可能会向各种方向起步，我们不太可能制作各个角度的起步动画，这个时候Database支持BlendSpace类型的动画就很有必要了，我们制作好BSWalkFwdLeftStart后，将NumberOfHorizontalSamples设置为20，表示横向生成多少个采样动画，20就意味着会生成20个采样动画, MirrorOption设置为OriginalAndMirrored可以生成Right版本的动画。

![](.\UE5MotionMatchingPracticePic/BlendSpace.png)


当角色向后背方向起步转身行走时，我们通过RewindDebugger发现这时候选择动画出现了一些问题，理想情况应该是始终播放WalkFwdStartL180这个动画，但实际情况是播放了一段WalkFwdStartL180后又选择了WalkFwdStartL135的动画，我们在RewindDebugger将时间轴拖到发生跳转的那一刻，看下发生了什么。

![](.\UE5MotionMatchingPracticePic/6.png)

![](.\UE5MotionMatchingPracticePic/7.png)

可以看到两者的Cost还是很近的（180L动画Cost为0.591, 135L动画Cost为0.507）180L吃亏在第三个Trajectory采样点的FaceDirection上，解决方案有多种

* 调整Trajectory采样时间，每个采样时间点权重不同等
* 修改Schema中的ContinuingPoseCostBias，在[《最后生还者2》中的Motion Matching](https://zhuanlan.zhihu.com/p/403923793)中我们已经讲过了Bias的含义了, 我们这里设置为-0.12
* WalkFwdStart系列动画中添加PoseSearchBlockTransitionNotifyState,将BeginTime设置为0.13，这样一旦选择好转身动画后几乎不会再反复横跳了

这里还有一个重大的改变就是Movement的运动全部基于SpringDamper而不是虚幻原生加速度，摩擦力，旋转速度那一套，为什么这么修改呢，SpringDamper有哪些好处呢？

* 对于旋转来讲，虚幻原生的RotationRate并不能反映真实情况，人在转身90或者180的时候并不是按照一个固定速度旋转的，SpringDamper要比RotationRate更合理
* 虚幻原生那套计算速度的函数CalcVelocity太麻烦了，不管对于策划还是程序，需要调整摩擦力，加速度，如果MaxWalkSpeed换了，这些值可能还得再调一遍，停步时还得调整摩擦力和减速度，SpringDamper相对来讲要好很多，策划一般仅调整一个参数即可
* 运动模型应该尽可能地简单，因为将来会开发Maya相关的插件用来检查动画和游戏运动模型是否很好地匹配，虚幻原生的运动模型显然没有SpringDamper简单

修改的话也比较简单，自定义MovementComponent和TrajectoryComponent, 对于MovementComponent来说，需要重载PhysicsRotation和CalcVelocity函数，我们可以在这两个函数中求出TargetRotation和TargetVelocity, 并且知道当前的CurRotation和CurVelocity, 调用QuaternionSpringInterp和VectorSpringInterp就能求出当前的Rotation和Velocity了。TrajectoryComponent需要重载StepPrediction和StepRotationWS函数，利用相同的方法实现SpringDamperSmooth，需要注意的是需要在PredictTrajectory调用前将MovementCompoent的SpringState同步给TrajectoryComponent的SpringState，保证预测更加准确。

{SpringDamper.mp4}

通过以上调整，我们再看下各方向起步的效果：

{SpringDamper+参数调整+NotifyState后的起步效果.mp4}

视频中有两个点可以说下
* 运动方向与角色朝向分别是0，90，135，180的时候，分别选择了相应的动画并播放，符合我们的预期；
* 特别有意思的是，我们提供的资源里面并没有Pivot相关的动画，但是视频后半部分角色在进行Pivot相关运动时，可以看到角色利用WalkStart动画展示出了Pivot效果，表现相当不错，也一点也展示出了MotionMatching系统的优势;

## 停步动画
添加停步动画时刚开始想法是使用一个WalkFwdStop_RU, 通过配置OriginalAndMirrored来生成左脚的动画，但运行时发现有问题，因为没有配套的Idle动画导致最后出现了旋转的问题，所以没有采用镜像来生成而是提供了两个停步动画WalkFwdStop_LU和WalkFwdStop_RU

{RightUpFoot镜像生成的LeftUpFoot在融合Idle会出现旋转的问题.mp4}

当我们选择停步时两脚的情况是不确定，MotionMatching给出的结果可能是WalkFwdStop_LU 0.8s开始播放，但是我们Movement停步计算出的时间几乎每次都是相同的，这将导致WalkStop动画的脚已经站定了但Movement还在移动或者相反的情况，如果是DataDriven或者有Clamp设置还好，但CodeDriven会经常出现滑步的问题，除了IK解决以外也可以考虑添加更多种类型的WalkStop动画(像《FF7重制版》做的那样)，但特别注意的是，如果不是配套的动画资源直接拿过来使用的话，会导致很多问题，比如下面视频里看到的，Idle不一致，这是个很严重的问题，CorePose始终应该是确定且唯一的

{随意找来的动画资源导致CorePose不一致.mp4}






转圈圈时动画不正常，添加WalkArcLoop动画后，正常了(Pivot表现怪异了，后续添加Pivot相关动画即可)
{添加WalkArcLoop前/后.mp4}




CollectGroupIndices计算Group是否存在问题？通过Group过滤Pose可以正常运行吗？没有问题，匹配时调用GetSourceAssetGroupTags直接拿到FPoseSearchDatabaseSequence，然后使用其中的GroupTags



使用MotionMatching需要面对的核心问题：
1. 动画数据需要RootMotion数据，在大量动捕数据(无根骨骼信息)的前提下，如何快速生成相对准确的RootMotion?
   https://twitter.com/Bobby_Anguelov/status/1539982427111702530

2. 动画师需要按照什么标准修改根骨骼数据以匹配Gameplay的需要，是自动化检测工具提醒还是其他?
3. CorePose要保持一致，比如Idle的状态，这时候动画师和技术需要定义好各种类型的CorePose，动捕数据需要大量粘贴CorePose
4. 虚幻资源分析工具或者Maya插件，对动画资源分析从而指导如何设置Movement参数或者反过来从Movement指导动画资源应如何调整，双向都支持的工具



将Idle和WalkFwdLoop放入到Loops Group中去，当输入稳定时，我们Bias在Loops Group中挑选数据
![Loops Group](.\UE5MotionMatchingPracticePic/1.png)

![可以看到DebugDrawQuery显示的Trajectory明显是歪的](.\UE5MotionMatchingPracticePic/2.png)
我们在AnimNode_MotionMatching中勾选DebugDraw和DebugDrawQuery,我们发现在角色快速转向后，Debug显示的Query跟预期的不太一样，刚开始怀疑是引擎有bug，后面查代码发现并不算bug，因为我们一开始会很自然的想到MotionMatching**每帧**都在查询最适合的帧并播放，其实并不是这样的，勾选DebugDrawQuery后，Debug显示的**上次**Search并且采用的那个Result中的QueryVector, 但并不是每一帧都会Search并采用Result的，AnimNode_MotionMatching中的SearchThrottleTime表明了不是每帧都会Search的(之前提到过，如果当前的动画已经到末尾了会强制Search)，即使执行了Search也不见得会使用Result，这依赖于这个Result是否足够合适，既然不是每帧都Search并采用Result，所以，Debug显示的Query始终为**上次**Search并且采用的Result的那个Query,所以会看到DebugDrawQuery和MotionTrajectory不一致的情况









RewindDebugger
* 蓝色显示QueryVector, 绘制PoseVector
* 红色显示**选中**的PoseIdx(如果在PoseDatabase中没有选中任何Pose则不显示), 绘制的是PoseIdx
* 绿色显示当前被MotionMatching认为最佳的PoseIdx(数量一般为1个，如果在ActivePose中没有选中则不显示), 绘制的是PoseIdx
* 灰色显示ContinuingPose的PoseIdx(数量一般为1个，如果在ContinuingPose中没有选中则不显示, 默认为非选中状态), 绘制的是PoseIdx

(可以看到在绘制时仅仅QueryVector向FDebugDrawParams传入的是PoseVector，其他的都是传入PoseIdx，因为调用Search时传入的QueryVector本身就是众多FeatureVector，而Database中存储的数据都是由PoseIdx组成的，每个PoseIdx都有自己的FeatureVectors, 所以当需要显示Database中某个Pose的调试信息时，用PoseIdx指代更加方便。不管是传入PoseVector还是PoseIdx，调用的都是DrawFeatureVector，最后都是调用到UPoseSearchFeatureChannel::DebugDraw，没有任何区别)



不同的FeatureChannel的绘制代码在UPoseSearchFeatureChannel::DebugDraw中实现，需要特别注意的是，绘制Trajectory时，LinearVelocity和FacingDirection很难区别，因为使用的是同一种颜色并且有些时候两个方向的长度还是等长的...

![](.\UE5MotionMatchingPracticePic/3.png)

通过UPoseSearchFeatureChannel_Trajectory::DebugDraw代码我们可以知道，绘制LinearVelocity时，向量的长度表示了速度的大小，而FacingDirection却始终是单位向量，所以可以得出结论，如果向量的长度等于圆球的半径大小那就是FacingDirection, 如果向量长度有长有短，那表示的就是LinearVelocity.


Cost[0]中的0表示ChannelIdx，指的是该Channel在Schema Channels中的索引值，对照着Schema的设置可以看到，Cost[0]指的是TrajectoryCost, Cost[1]指的是PoseCost


![](.\UE5MotionMatchingPracticePic/4.png)

![](.\UE5MotionMatchingPracticePic/5.png)

建议：
* DrawDebugString貌似在RewindDebugger中无效
* FeatureVector 63各个含义？
* Database Feature抖动？
* FacingDirection 和 LinearVelocity可以用颜色区分下
* Cost计算是否有bug，至今看不出为什么会选择135而不是180？
* 有Flag能够标识出当前是否正在查询
* PoseDatabase中并不会表示BlockTranstion的标识
* Idle情况下没有考虑ErrorTolerance的情况
* 性能问题
* Steering和Clamping有什么计划呢


你好，我使用了PoseSearch一段时间，期间发现了一些问题，希望能讨论下

1. DrawDebugString貌似在RewindDebugger中无效，导致DrawBoneNames和DrawSampleLabels不可用，是已知问题吗？
2. (配图) 每个索引指的是哪个Channel？ 是Position.X，还是FaceDirection.Y？ 是不是可以考虑用ToolTips来帮助开发者？
3. (配视频)当我在Database拖动时间轴时，发现Feature会有抖动，我怀疑是插值导致的问题，这符合预期吗？
4. (配图)建议FacingDirection和LinearVelocity可以用颜色区分下，有的时候很难区分
5. (配图)不知道Cost计算是否有bug，目前看不出MotionMatching系统为什么会选择135的动画而不是180？
6. 有没有考虑在MotionMatchingState中添加一个属性表示是否调用过Search函数
7. PoseDatabase中并没有列属性表示BlockTranstion的标识，这会导致一个问题是某个Pose Cost最小但是不能跳转
8. (配代码)Cost比较没有考虑ErrorTolerance的问题，导致一个问题是当角色静止不动时Pose也在跳转
9. (配图)Steering和Clamping有什么计划呢
