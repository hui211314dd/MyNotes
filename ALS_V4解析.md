# Ragdoll

![Ragdoll动画蓝图](./ALSV4Pic/1.png)

可以看到只有当MovementState为Ragdoll时才会走Ragdoll States.

Ragdoll蓝图部分的代码主要在ALS_Base_CharacterBP的Ragdoll System页签下.

![Ragdoll蓝图代码](./ALSV4Pic/2.png)

简单来说就是启用Ragdoll(X键)时，调用RagdollStart(碰撞设置，启用物理以及关闭所有蒙太奇)，RagdollUpdate是在Ragdoll期间执行的工作(包括更新速度信息，物理模拟设置，更新重力设置以及调用SetActorLocationDuringRagdoll等)，SetActorLocationDuringRagdoll保障胶囊体不要脱离Mesh太多，否则相机不会跟随Mesh，同时避免胶囊体嵌到地面中；结束Ragdoll时(再次X键)会调用RagdollEnd,除了会执行RagdollStart中相反的操作以外，会调用SavePoseSnapshot以及播放起身的Montage

* SavePoseSnapshot的作用是当关闭物理到播放起身Montage这个期间能够平滑播放；当关闭物理那一刻Mesh的Pose为PoseA,关闭物理时动画蓝图会完全接管，紧接着播放起身Montage，比如BlendInTime是0.2，如果不做任何处理，在这0.2s期间，起身动画可能会与其他动画进行融合，但我们希望的是跟PoseA融合，所以我们可以通过SavePoseSnapshot把那一帧的Pose保存起来并用于融合；

![Pose Snapshot的使用](./ALSV4Pic/3.png)

* 会根据Ragdoll脸的朝向以及Overlay播放不同的起身动画, 你可以看到配置的所以起身动画都是起到半蹲状态的，如果在执行Ragdoll之前是站立状态，则Stance为Standing, 半蹲则为Crouthing, 当播放起身动画后，动画状态机会执行MainGroundedStates, 如果执行Ragdoll之前是Standing状态，播放起身动画后，状态机会依次走(CLF)Crouching LF-> (C)->(N)->(CLF)->(N)Transition, 所以可以看到GetUp->Crouch->Stand的变化。

![MainGroundedStates](./ALSV4Pic/4.png)

还有一个细节就是攀爬的过程中如果Ragdoll的话，需要把MantleTimeline Stop掉，否则会出很多问题，OnMovementStateChanged就是来处理这类问题的(也可以说ALS_Base_CharacterBP中的On***Changed都是处理这类状态切换时需要特殊处理的问题的)。

![OnMovementStateChanged](./ALSV4Pic/5.png)

进一步的细节可参考资料：

[布娃娃系统-ALS V4实现方案详解](https://zhuanlan.zhihu.com/p/383134963)

[UE4高级运动系统（Advanced Locomotion System V3）插件分析](https://zhuanlan.zhihu.com/p/101477611)

