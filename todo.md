1. 基于TargetPawn的CameraAnim
2. Virtual Bone
3. Control Rig
4. Animation Insights
5. 《Game Anim》
6. https://rockhamstercode.tumblr.com/
7. Maya动画制作
8. ALS5系统
9. 各种插值算法，包括Critically Damped Spring，inertialization等



明年计划：
1. maya动画学习，没有什么前置知识，而且学完后可以自己做动画进行其他功能试验。
2. 动画蓝图的实践，比如ALS这种系统，要经常实践，否则经常处于纸上谈兵的怪圈，无前置知识，需要文档总结, 很多基础系统的学习包括同步组，Pose Driver，AnimationLink等等
3. 
4. 国外文档的翻译，接触下最新的理论或者最佳实践等。
5. 物理动画，需要大量数学的前置知识，所以过看边学，不要先系统去学数学，有需要再去学。
6. 引擎的具体流程，从动画到drawcall。







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
. 各个模块文档记录(Trajectory, 添加TrajectoryPoint新成员需要修改哪些内容, Mirror原理, Data Normalize, )  
. MMConfig能否减少被依赖
. stride scaling



本周计划：
1. Doom - NPC-Strafe问题
2. SpringDamper，各种Lerp, FInterpTo 惯性化插值比较以及关卡

每日任务：
1. Demo工程需要分析看下。
2. SpringDamper需要掌握。
3. 




MotionMatchingDemo问题
1. bone_angular_velocities的理解以及骨骼速度和角速度计算原理？
2. range-start range-stop和 framesNum的关系？