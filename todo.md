1. 基于TargetPawn的CameraAnim
2. Virtual Bone
3. Control Rig
4. Animation Insights
5. 《Game Anim》
6. https://rockhamstercode.tumblr.com/
7. Maya动画制作
8. ALS5系统
9. 各种插值算法，包括Critically Damped Spring，inertialization等


近期工作：
1. Demo中的IK(先自己梳理下流程然后再看代码)和generate_database.py
2. UE5 PoseSearch工程
3. Maya学习
4. 数学关卡


MotionMatchingDemo问题
1. bone_angular_velocities的理解以及骨骼速度和角速度计算原理？
2. range-start range-stop和 framesNum的关系？
3. bone_positions 和 db.bone_positions(frame_index) 关系？
4. transition_src_position,transition_src_rotation,transition_dst_position,transition_dst_rotation以及
   bone_offset_positions, bone_offset_velocities, bone_offset_rotations, bone_offset_angular_velocities含义以及关系？
5. inertialize_pose_transition中inertialize_transition为什么传入的都是root_position？


重点：学习inertialize时掌握速度，速度offset和位置的关系？



Footlock的问题
1. 先自己梳理下流程，问题然后再看代码
2. 如果没有看代码的话，可能会想到动画里面添加Notify表示左右脚落下抬起，但这个工作量多且繁杂，所以不会优先考虑
3. 首先面对的问题就是如何判断某个脚落下了或者抬起了？ Trace或者判断骨骼位置？
4. 如果是后者的话碰到凹凸或者台阶如何处理(仔细想想，Footlock貌似不处理这种问题，Footlock大致原理应该是发现动画脚骨骼落下不动了判定为落下并且锁住，台阶和凹凸问题归Simulation管)
5. 大致的思路应该是记录左右脚落下的时间点以及位置，然后锁住，直到另外一只脚也落下，这个时间段使用Two Bone IK把脚一直锁住在指定位置



PoseSearch的问题
1. PoseSampleTimes怎么回事，不是只需要CurPose就可以吗？
2. 需要搞明白Character生成的Trajectory与Mesh的转换关系(转90的问题)
3. Database，Schema,MetaData之间的联系以及参数含义需要搞的明明白白
4. 如何调试？


明年计划：
1. maya动画学习，没有什么前置知识，而且学完后可以自己做动画进行其他功能试验。
2. 动画蓝图的实践，比如ALS这种系统，要经常实践，否则经常处于纸上谈兵的怪圈，无前置知识，需要文档总结, 很多基础系统的学习包括同步组，Pose Driver，AnimationLink等等
3. 国外文档的翻译，接触下最新的理论或者最佳实践等。
4. 物理动画，需要大量数学的前置知识，所以过看边学，不要先系统去学数学，有需要再去学。
5. 引擎的具体流程，从动画到drawcall。
6. UE5 nes插件 base SimpleNES


4月目标:
PoseSearch看完 PoseSearch系列完成MotionMatching, Weight调整

微积分完结
Maya新手课完结
四元数完结


1. PoseMatching文档（完成！）
2. 数值分析(需要的时候再看)
3. Mirror,Footlock以及MotionMatchingDemo分拆文档
4. ALS以及PoseSearch看完
5. 剖析RootMotion以及AnimNode各个函数含义



各种指针:

TObjectPtr

TSoftObjectPtr

TWeakObjectPtr

TSharedPtr

TSharedFromThis


虚幻引擎必知项:
AnimSubSystem ???

序列化与序列化，编辑器编辑与Runtime的关系
反射系统以及UBT
FProperty原理以及各个设置
GC原理以及优化
智能指针
网络同步
TaskGraph使用以及原理
TickGroup使用以及原理
Movement的同步以及原理

TraceLog/UE_TRACE_LOG 配合PoseSearch RewindDebugger

4. CreateTickRecordForNode/SyncScope.AddTickRecord做了哪些事情以及流程？
TODO Sync原理

5. URO原理以及流程？
TODO URO使用以及原理

MotionWarping, DeltaCorrection原理以及重要性，需要举例，比如怪物跳起攻击

DistanceMatching的疑惑(通过Distance反推后的Pose与当前Pose差距过大)以及代码如何解决的