将Idle和WalkFwdLoop放入到Loops Group中去，当输入稳定时，我们Bias在Loops Group中挑选数据
![Loops Group](.\UE5MotionMatchingPracticePic/1.png)

在Database设置中出现了一个问题，假如有一个长段的walk动画，但是首尾都是一些TPose的动画，我们希望在设置SampleRange时截断他们，并且Range.Min和Range.Max帧相同，当时希望播放到最后帧Range.Max后紧接着播放Range.Min，为了支持这个功能，我们特意将该动画的Loop设置为true，但是事与愿违，发现播放帧的顺序为1，2，3，4，5，6，5，6，5，6.....播放到末尾帧后进行了查询，发现第5帧为理想跳转帧，后面5，6一直循环，我们并不希望这样，因为放到Loop Group的初衷就是循环采样，通过代码(FMotionMatchingState::CanAdvance)我们发现SampleRange截断后再Advance为false，这样会紧接着调整到Search的结果上,因此会造成上面的5,6循环问题。所以解决方案是我们导入引擎前就要保证这个动画是循环动画并且首尾衔接，这样就不要设置SampleRange。如何理解呢？AnimSequence中的Loop表示这个动画本身为首尾衔接的Loop动画，导入到Database中设置了SampleRange，这时候不能指望Range内的动画依然可以首尾衔接

{稳定输入下WalkLoop循环播放.mp4}





![可以看到DebugDrawQuery显示的Trajectory明显是歪的](.\UE5MotionMatchingPracticePic/2.png)
我们在AnimNode_MotionMatching中勾选DebugDraw和DebugDrawQuery,我们发现在角色快速转向后，Debug显示的Query跟预期的不太一样，刚开始怀疑是引擎有bug，后面查代码发现并不算bug，因为我们一开始会很自然的想到MotionMatching**每帧**都在查询最适合的帧并播放，其实并不是这样的，勾选DebugDrawQuery后，Debug显示的**上次**Search并且采用的那个Result中的QueryVector, 但并不是每一帧都会Search并采用Result的，AnimNode_MotionMatching中的SearchThrottleTime表明了不是每帧都会Search的(之前提到过，如果当前的动画已经到末尾了会强制Search)，即使执行了Search也不见得会使用Result，这依赖于这个Result是否足够合适，既然不是每帧都Search并采用Result，所以，Debug显示的Query始终为**上次**Search并且采用的Result的那个Query,所以会看到DebugDrawQuery和MotionTrajectory不一致的情况





BlendSpace存在bug，没有办法，只能把数个WalkStartL添加到Database中并且同时生成Mirrored数据，这里顺便给上面的WalkLoop也生成了Mirrored数据，再次运行游戏，发现角色走路过程中一直在JumpPose导致脚部有滑步

{WalkLoop启用Mirrored后发现即使稳定输入下也在反复切换Pose.mp4}

我们通过RewindDebugger发现切换的原因是MirroredWalkLoop的某一个Pose Cost值为0.001，而ContinuityPoseCost为0.003，而(0.003 - 0.001)/0.003 = 2/3 > 40%, 所以系统认定MirroredWalkLoop的Pose比ContinuityPose更加适合，所以选择了跳转。如此反复导致了Pose的频繁切换，如何解决呢？一种可行的解决方法是增加MirroringMismatchCost,这样如果Origin动画和Mirrored动画相互切换时就产生了额外的Cost,我们设置为0.02即可。设置后我们发现稳定输入的情况下，不再频繁切换Pose了~



RewindDebugger
* 蓝色显示QueryVector, 绘制PoseVector
* 红色显示**选中**的PoseIdx(如果在PoseDatabase中没有选中任何Pose则不显示), 绘制的是PoseIdx
* 绿色显示当前被MotionMatching认为最佳的PoseIdx(数量一般为1个，如果在ActivePose中没有选中则不显示), 绘制的是PoseIdx
* 灰色显示ContinuingPose的PoseIdx(数量一般为1个，如果在ContinuingPose中没有选中则不显示, 默认为非选中状态), 绘制的是PoseIdx

(可以看到在绘制时仅仅QueryVector向FDebugDrawParams传入的是PoseVector，其他的都是传入PoseIdx，因为调用Search时传入的QueryVector本身就是众多FeatureVector，而Database中存储的数据都是由PoseIdx组成的，每个PoseIdx都有自己的FeatureVectors, 所以当需要显示Database中某个Pose的调试信息时，用PoseIdx指代更加方便。不管是传入PoseVector还是PoseIdx，调用的都是DrawFeatureVector，最后都是调用到UPoseSearchFeatureChannel::DebugDraw，没有任何区别)



不同的FeatureChannel的绘制代码在UPoseSearchFeatureChannel::DebugDraw中实现，需要特别注意的是，绘制Trajectory时，LinearVelocity和FacingDirection很难区别，因为使用的是同一种颜色并且有些时候两个方向的长度还是等长的...

![](.\UE5MotionMatchingPracticePic/3.png)

通过UPoseSearchFeatureChannel_Trajectory::DebugDraw代码我们可以知道，绘制LinearVelocity时，向量的长度表示了速度的大小，而FacingDirection却始终是单位向量，所以可以得出结论，如果向量的长度等于圆球的半径大小那就是FacingDirection, 如果向量长度有长有短，那表示的就是LinearVelocity.



DrawDebugString貌似在RewindDebugger中无效



Cost[0]中的0表示ChannelIdx，指的是该Channel在Schema Channels中的索引值，对照着Schema的设置可以看到，Cost[0]指的是TrajectoryCost, Cost[1]指的是PoseCost


![](.\UE5MotionMatchingPracticePic/4.png)

![](.\UE5MotionMatchingPracticePic/5.png)
