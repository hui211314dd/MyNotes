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

# OverlayLayer

## Default

Default为默认的Overlay动画，它定义了一个很标准的Overlay模板，基本上点出了其他Overlay动画应该考虑哪些参数。

![Overlap标准的动画蓝图样子](./ALSV4Pic/6.png)

Idle/MovementPoses考虑了角色在不同Gait(Idle/Walk/Run/Sprint)以及姿态(Stand/Crouching)的Pose表现

InAirPoses考虑到了在空中的事情

SecondaryMotion叠加了呼吸动画

可以看到ALS_StanceVariation_Normal中Layering_***和Layering_
***_Add都为1，表示最后希望给Overlay上叠上全部的Locomotion效果。

## Masculine(男性化)

跟Default一致，ALS_StanceVariation_Normal中Layering_***和Layering_
***_Add都为1，表示最后希望给Overlay上叠上全部的Locomotion效果。

## Feminine(女性化)

Feminine的结构跟Default一致，但Curves值不太一样。

![Arm做了部分叠加](./ALSV4Pic/7.png)

Layering_Arm_L_Add和Layering_Arm_R_Add的数值不是1而是0.75，我们先看下如果改成1后的效果：

{Arm_Add为1.mp4}

可以看到当疾跑时角色女性化的手臂动画和疾跑手臂甩动动画合并效果不理想，我们希望手臂甩动幅度小一些即叠加的Locomotion效果减一些，所以把Layering_Arm_L_Add和Layering_Arm_R_Add的数值改成了0.75，而其他数值不变，所以仅仅改变了手臂的叠加效果。

{Arm_Add为0.75.mp4}

## HandsTied(双手被捆)

HandsTied的结构跟Default一致，但Curves值不太一样。

![Arm不做任何叠加即只使用Overlay](./ALSV4Pic/8.png)

双手被捆的情况下，不管是什么Gait情况下，双手的状态应该是始终保持不变，即双手的状态不受Locomotion影响，所以需要将Layering_Arm_L_Add和Layering_Arm_R_Add的数值改成0，即Arm相关的动画只使用Overlay，不做任何叠加

## Injured(受伤)

Injured的结构跟Default一致，但Curves值不太一样。

![ArmL不做任何叠加即只使用Overlay而ArmR是全套叠加](./ALSV4Pic/9.png)

左腹受伤的情况下，ArmL应该只使用Overlay，不会叠加甩臂的动画，所以Layering_Arm_L_Add为0，Layering_Arm_R_Add为1

## Box(搬箱子)

一般情况下，我们第一直觉会是跟HandsTied一样的设计，我们看下如果跟HandsTied设置一样会出现什么问题。

{Box只设置Arm.mp4}

很明显的问题是左手没有启用IK, 还有一个不是很明显的问题，如下图：

![埋头苦干](./ALSV4Pic/10.png)

可以看到在搬箱子疾跑状态下，头和身体的姿态不太合理，头部和身体应该挺起来一些，而不是“埋头苦干”，可以调整Laying_Head_Add和Layering_Spline_Add的数值

![挺直身子](./ALSV4Pic/11.png)

我们再看下最终的效果:

{Box设置ArmHead以及Spline.mp4}

跟Defualt相比，下面的Curves的值不太一样:

Enable_HandIK_L 1

Layering_Arm_L_Add 0

Layering_Arm_R_Add 0

Layering_Head_Add 0.5

Layering_Spline_Add 0.75


Box的结构跟Default略有不同，可以看到Box考虑到了攀爬，翻滚和Ragdoll起身的特殊情况

![Box动画蓝图](./ALSV4Pic/12.png)

> _去掉关于Overlay Override State的判断也不怎么影响表现，所以不再解释_

添加Enable_HandIK_L Curve后左手IK会自动生效，但有意思的是，项目中没有写任何有关的IK逻辑比如左手应该在哪个位置，IK是如何知道目标位置的呢？这其实涉及到一个概念“SpaceSwitch”即空间转换，我们必须得搞明白为什么出现了左手脱离箱子的情况，子骨骼的位置是通过从root到该骨骼依次相乘得到的而且动画师只在一些帧上key动画，这就导致中间的部分都是插值得到的(hips会插值，spline会插值，arm会插值等等)，这些插值后的骨骼再依次相乘得到的位置可能会跟理想位置有些出入，这就导致了抖动的问题。而虚拟骨骼就是为了解决这类问题的。

关于虚拟骨骼的参考资料：

[虚幻引擎新增动画特性](https://www.bilibili.com/video/BV1UE41147jS/?share_source=copy_web&vd_source=e00408fe4d52d25a32600d385dd6b894)

## 









# LayerBlending

* Layering_***表示是否需要融合Overlay相关部位的动画，0表示不融合，完全使用BaseLayer中的动画，大于0表示最终效果可能需要Overlay和BaseLayer的综合效果(记住并不是表示完全使用Overlay)，具体比例得看下面的参数

* Layering_Arm_L_Add表示将(BaseLayer-BasePose)这个叠加动画叠加到Overlay的程度，如果是1表示最终动画需要无删减的显示Overlay和BaseLayer的综合效果；如果是0表示完全使用Overlay的效果，BaseLayer的中这个部位的效果被舍弃。


常见场景：
如果Layering_***为0，则Layering_
***_Add参数不考虑，表示直接使用BaseLayer的效果，调试模式下显示为黑色($\color{red}{TODO 举例}$)

如果Layering_***为1，Layering_
***_Add为0，表示完全使用Overlay的效果，调试模式下显示为白色($\color{red}{TODO 举例}$)


如果Layering_***为1，Layering_
***_Add为1，表示是Overlay和BaseLayer的综合效果，调试模式下为红色($\color{red}{TODO 举例}$)

($\color{red}{如果两个数值都是大于0小于1的时候时，需要举例例子说明什么情况下调整哪个数值，毕竟两个数值都可以表示融合的程度}$)


参考资料：


[UE4分层混合节点LayeredBlendPerBone设置](https://zhuanlan.zhihu.com/p/428242048)

# BaseLayer

# FootIK

# HandIK

# Curves解释

Weight_Gait, 0表示Idle, 1表示Walk(包括半蹲或者站立)，2表示Run, 3表示Sprint

Weight_InAir, 0表示地面上，1表示在空中


# Others

Overlay Override State: 

* 0: 默认

* 1：Mantle(1m)攀爬

* 2: Rolling翻滚

* 3: Rolldog后GetUp起身


# ModifyCurve

Grounded  EnableFootIKL 1.0, EnableFootIKR 1.0

Fall BasePoseN 1.0, WeightInAir 1.0

Jump BasePoseN 1.0, WeightInAir 1.0

Land EnableFootIKL 1.0, EnableFootIKR 1.0, FootLockL 1.0, FootLockR 1.0, BasePoseN 1.0

LandMovement EnableFootIKL 1.0, EnableFootIKR 1.0, BasePoseN 1.0

Standing BasePoseN 1.0

Crouching WeightCrouching 1.0, BasePoseCLF 1.0

FromRoll FootLockL 1.0, FootLockR 1.0

(CLF)NotMoving FootLockL 1.0, FootLockR 1.0, EnableTransition 1.0, RotationAmount RotationScale

(CLF)RotateLeft RotationAmount RotateRate

(CLF)RotateRight RotationAmount RotateRate

(CLF)Stop FootLockL 1.0, FootLockR 1.0

(N)NotMoving FootLockL 1.0, FootLockR 1.0, EnableTransition 1.0, RotationAmount RotationScale

LockLeftFoot FootLockL 1.0

LockRightFoot FootLockR 1.0

PlantLeftFoot FootLockL 1.0

PlantRightFoot FootLockR 1.0

(N)RotateLeft90 RotationAmount RotationScale

(N)RotateRight90 RotationAmount RotationScale

Torch LayeringArmLAdd 0.0

Box LayeringSpineAdd 0.5, LayeringHeadAdd 0.5

Rifle LayeringArmLAdd 0.0, LayeringArmRAdd 0.0

Bow LayeringArmLAdd 0.5

(N)MoveF/(N)MoveB/(N)MoveRF/(N)MoveRB/(N)MoveLF/(N)MoveLB YawOffset FYaw

(CLF)MoveF/(CLF)MoveB/(CLF)MoveRF/(CLF)MoveRB/(CLF)MoveLF/(CLF)MoveLB YawOffset FYaw

