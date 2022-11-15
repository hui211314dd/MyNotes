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
3. 插入到story中，点击Ghost先设置到0，0点上，将Hips Trajectory打开，随后使用顶视角微调Ghost的位置，保证Hips的路线能够对齐到轴线上(即使旋转45度的起步动画也要对齐到轴线上)，旋转角色面向指定方向，比如-Z(可能出现偏差，没关系，后面会校正)
4. Story中插入Loop动画(比如walk/idle)，对齐融合
5. (可选：动画Range微调)Frame SetBegin/End，然后Bake到新的Take中，选中HipsControl，删除过渡点的关键帧
6. 新建Layer，粘贴CorePose，处理矫正问题(比如期望是转身45度但实际30度)，滑步等问题，依赖AdjustmentBlend工具
7. 激活用于生成RootMotion的约束，BakeToControlRig，然后选中Root,点击BakeSelected,这样就生成好RootMotion数据了，去掉约束的激活
8. 编辑RootMotion曲线，尽可能的平滑，匀速旋转或者位移可以使用线性插值，最重要的是准确地覆盖，Root点要尽可能在两腿之间，否则会出现脚部打滑的问题
9. Bake到Skeleton, 然后选中所有骨骼以及Character导出成动画文件


WalkSpeed: 122
JogSpeed:
SprintSpeed:



动画插入stroy前先Bake到ControlRig上
保存Pose前记得先Bake到ControlRig上
Import Motion就是将导入fbx中的骨骼数值设置到当前Character的骨骼上，所以导入动画后Source选择None

todo base_env需要定义好Relation约束，Root属性约束以及缩小网格线

