1. 基于TargetPawn的CameraAnim
2. Virtual Bone
3. Control Rig
4. Animation Insights
5. 《Game Anim》
6. https://rockhamstercode.tumblr.com/
7. Maya动画制作
8. ALS5系统
9. 各种插值算法，包括Critically Damped Spring，inertialization等


学习方法总结：
1. 当某些代码跟你预想不一样的时候，试着想想那些你以为是这个意思的变量真的是这个意思吗？！试着打印出来是否符合你的预期
2. 学习就像爬山，当你学习某样知识的时候，如果觉得轻松，其实别人都是一样的，没有什么了不起的，但是，当你感觉到吃力的时候，同样地，大部分人学习的时候也是吃力的，如果你能坚持下来，那你最起码在技术上有了成长
3. 学习代码的时候，第一步先能运行并且可调试，并且学习计划得有，比如今天看哪个模块或者哪个知识点，漫无目的看代码效率会很低
4. 看代码的主要目的是更加深刻认识原理，作用，使用场景，注意事项等，看了不知道用在哪儿没啥意义
5. 


明年计划：
1. maya动画学习，没有什么前置知识，而且学完后可以自己做动画进行其他功能试验。
2. 动画蓝图的实践，比如ALS这种系统，要经常实践，否则经常处于纸上谈兵的怪圈，无前置知识，需要文档总结, 很多基础系统的学习包括同步组，Pose Driver，AnimationLink等等
3. 国外文档的翻译，接触下最新的理论或者最佳实践等。
4. 物理动画，需要大量数学的前置知识，所以过看边学，不要先系统去学数学，有需要再去学。
5. 引擎的具体流程，从动画到drawcall。
6. UE5 nes插件 base SimpleNES







. Trajectory上添加上Dir和TurnSpeed(完成)
. 完成Mesh的空间转换以及debug(需要确认下插件目前接收的是哪个空间坐标系的) (完成)
. 现有的模式下能动起来！(完成)
. Pose Matching相关(完成)
. Trajectory相关配置应该继续放在MMConfig中去。(完成)
. Trajectory数据坐标轴换成统一的(完成)






. Capsule与Entity的分离以及Adjustment Clamp的实现(重要！)
. blend data-driven and code-driven(不用做)
. Debug工具(重要！)
. 熟悉Cost算法(主要能够多维度控制，Tag，Bias，Group等等)(重要！)

. Sample上添加Velocity和TurnSpeed数据
. 解决滑步问题(IK等)
. 同步以及其他问题(RootMotion, Motion Wrap)
. 各个模块文档记录(Trajectory, 添加TrajectoryPoint新成员需要修改哪些内容, Mirror原理, )  
. MMConfig能否减少被依赖
. stride scaling


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

MotionTrajectory
1. EffectiveTimeDomain干啥用的？
2. SampleRate有什么必要性呢？

重点：学习inertialize时掌握速度，速度offset和位置的关系？






