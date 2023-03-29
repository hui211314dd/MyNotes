# UE5中的MotionMatching-实践(二)

{导入动捕数据后的表现.mp4}

1. Idle选择不合理
2. 没有走直线的Walk
3. 要不频繁切换动画(Animation Shredding, 动画虽然没有抖动，但是过渡blend导致失去了很多动画细节)，要不Transition时动画一直不切换

如何解决：
* Idle制作成循环动画，并且选好CoreIdlePose
* 制作直线Walk循环动画
* 长动画做成短动画，用CorePose粘贴到Start位置，尾部跟上Idle/Walk循环动画
* 分组，并且动态调整各个Group的Weight


编辑动画时StartFrame不要有太多静止帧，否则会出现Search错误，比如WalkStart_45R如果开始站立时间太长，会导致停步时搜索到该动画的起始帧(理想情况应该是Idle)。

编辑动画的步骤:
1. 配置好环境，包括导入IdlePose, Loop动画(Walk,Jog,Sprint,Idle等)，定义好标准比如角色首帧Pose位置以及朝向。(特别重要，否则会出现PoseCost不符合预期的问题)
2. 导入动画资源，选好片段后SetStart,SetEnd，然后BakeToControlRig
3. 插入到story中，新建空Take, 点击Ghost先设置到0，0点上，先执行一次Frame SetBegin/End，将HipsEffector Trajectory打开，随后使用顶视角/前视角微调Ghost的位置(不一定非得是0,0点)，保证Hips的路线/两脚的中心点能够对齐到轴线上(即使旋转45度的起步动画也要对齐到轴线上)，旋转角色面向指定方向，比如-Z(可能出现偏差，没关系，后面会校正)
4. Story中插入Loop动画(比如walk/idle)，对齐融合
5. (可选：动画Range微调)Frame SetBegin/End，然后Bake到新的Take中，选中HipsControl，删除过渡点的关键帧
6. 新建Layer，粘贴CorePose，处理矫正问题(比如期望是转身45度但实际30度)，滑步等问题，依赖AdjustmentBlend工具
7. 激活用于生成RootMotion的约束，BakeToControlRig，然后选中Root,点击BakeSelected,这样就生成好RootMotion数据了，去掉约束的激活
8. 编辑RootMotion曲线，尽可能的平滑，匀速旋转或者位移可以使用线性插值，最重要的是准确地覆盖，Root点要尽可能在两腿之间，否则会出现脚部打滑的问题
9. Bake到Skeleton, 然后选中所有骨骼以及Character导出成动画文件


特别重要：在第4步插入动画对齐融合时特别小心，插入动画的RootMotion数据要与Loop本身的动画完全一致，否则会出现PoseCost过大匹配不上，或者即使匹配上，动画会有横行或者纵向的抖动。配图说明：

Transitions动画的Head和Tail如何处理：
WalkStart的话，Head应该尽可能短(不要一帧，否则首帧计算出的速度值有问题，当然也可以设置LeadIn动画), Tail同样不建议直接一个Pose，可以接一段短的Loop动画，不要太长，如果PastTrajectory设置的是0.4s, 那么Tail长度0.5即可(前提是导入引擎后，WalkStart的FollowUp设置了WalkLoop？？？)，为什么这样呢？比如当前动画播放的是WalkStart的最后一帧，这时候需要Search查找下一个动画了，但是PastTrajectory还处于WalkStart加速的一个状态，整个Trajectory看起来不是一个稳定前进的形状，所以不会找到WalkLoop的动画，如果Tail长度足够长，Trajectory形状足够稳定了，所以会很自然找到WalkLoop。还有一个原因就是Tail仅仅一个Pose只能保证PosePosition相同，不要保证Velocity以及Phase相同。

是否要使用引擎的LeadIn和Followup，这其实决定了动画制作的流程，不建议混用，使用LeadIn和FollowUp的话，动画师只需制作核心片段即可，但是要求与LeadIn和FollowUp动画首尾衔接，否则Foot的Velocity或者Phase计算有误；不使用LeadIn和FollowUp的话，动画师制作更为灵活，但修改Loop动画时修改起来比较麻烦。

WalkStop的话，Head动画可以略微长点(防止Velocity和Phase匹配不上)，其他以上。

Loop动画制作时要特别小心，特别是给Transition插入使用时，最好选取一个Cycle供使用，比如真正的WalkLoop有两个Cycle, 某些动画使用了前面的Cycle，另外一些动画使用了后面的Cycle, 这样混乱使用会导致Pose比较时不一致. 

如果可以制作工具帮助插入Loop并且生成固定好的RootMotion数据最好了，因为每个动画都单独为Loop插入动画生成Root数据的话，会有差异。


两个问题：
1. MB中明明RootMotion数据相同，L1数据与R1连接时有位移
2. L1结束时Search为什么Idle的PoseCost那么高呢


WalkSpeed: 122，4.074/frame
JogSpeed:
SprintSpeed:

WalkStartR:
DPS和位置改变曲线尽可能相同
45: 40帧，1.333s, DPS = 33.758， Distance: 95.84
90: 42帧，1.4s,   DPS = 64.2857, 
135:45帧，1.5s,   DPS = 90

180:60帧，2s,     DPS = 90--弃用

IDLEPose：
RightFoot：X:18.82，Y:13.86
LeftFoot：X：-23.70，Y:14.06


动画插入stroy前先Bake到ControlRig上
保存Pose前记得先Bake到ControlRig上
Import Motion就是将导入fbx中的骨骼数值设置到当前Character的骨骼上，所以导入动画后Source选择None

Group Weight机制，目前不支持

todo
使用DPS预测动画，Rotation由AnimationDriven, 通过Spring进行Filter
添加Circle动画，按住半径不同，制作LoopCircle动画