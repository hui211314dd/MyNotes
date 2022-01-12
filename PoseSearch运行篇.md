UE5中MotionMatching(二) 创建可运行的PoseSearch工程

## 前言
本篇主要讲述的是手把手创建一个可以运行的PoseSearch工程，并不会讲代码的实现细节，代码部分的分享在后面的系列文章再说。工程能够跑起来对于后续功能调试以及代码细节的理解会大有裨益~

我的UE5-Main是在2022.1.4日更新，而且PoseSearch(UE5对MotionMaching的称呼)本身就处于试验阶段，所以不保证将来是否会有大的改动。

PoseSearch插件路径：UnrealEngine\Engine\Plugins\Experimental\Animation\PoseSearch

如果你对Motion Matching感兴趣，可以看下我的其他文章。

[Motion Matching 中的代码驱动移动和动画驱动移动](https://zhuanlan.zhihu.com/p/432663486)

[《荣耀战魂》中的Motion Matching](https://zhuanlan.zhihu.com/p/401890149)

[《最后生还者2》中的Motion Matching](https://zhuanlan.zhihu.com/p/403923793)

[《Control》中的Motion Matching](https://zhuanlan.zhihu.com/p/405873194)

[游戏开发中的Pose Matching](https://zhuanlan.zhihu.com/p/424382326)

[MotionMatching中的DataNormalization](https://zhuanlan.zhihu.com/p/414438466)

[UE5中MotionMatching(一) MotionTrajectory](https://zhuanlan.zhihu.com/p/453659782)


## 开始吧！
1. UE5创建Third Person的工程，Blueprint/C++都可以，建议创建C++，主要是将来调试代码方便~
   
2. 启用Animation中的MotionTrajectory和PoseSearch插件，重启
   ![pic](.\PoseSerachRunPic/1.png)

3. Content Browser右键Miscellaneous/Data Asset,创建两个数据资源，分别是PoseSearchDatabase和PoseSearchSchema, 两个资源分别命名为PSLocomotionDB和PSLocomotionSchema
   ![pic](.\PoseSerachRunPic/2.png)

4. 导入动画资源，我这里使用的是商城里面的动画资源，动画资源必须带有根骨骼运动。如果手边没有可用资源的话可以用后面`参考资料`提到的育碧开源动画库
   ![pic](.\PoseSerachRunPic/3.png)

5. 打开PSLocomotionDB，Schema设置为PSLocomotionSchema, 点击Sequences右边的加号，将上一步导入的每个资源都添加进去，如果有n个动画资源，则添加n个，将来动画蓝图中的MotionMatching节点搜索动画帧时就是从这些动画里面查找的。PoseSearchDatabase有很多的参数设置，其中包括极其重要的Weight调整,因为我们仅仅是功能演示，所以这些参数先使用默认。设置如下：
   ![pic](.\PoseSerachRunPic/4.png)
   
   ![pic](.\PoseSerachRunPic/5.png)

6. 打开PSLocomotionSchema，Skeleton设置成我们的小白人UE4_Mannequin_Skeleton, Bones添加两根骨骼，分别是foot_l和foot_r, 表示我们在Pose Matching时仅仅对左右脚的骨骼信息感兴趣; PoseSample Times添加三项，分别设置为-0.1, 0.0, 0.1; Trajectory Sample Times添加六项，分别设置为-0.1, 0.0, 0.1, 0.2, 0.3, 0.4，这六项在Trajectory Matching时使用; 勾选UseFacingDirections, 其他参数默认。设置如下：
   ![pic](.\PoseSerachRunPic/6.png)

7. 第4步时我们导入了若干动画资源，现在我们编辑下这些动画资源。打开动画资源，属性栏里找到Meta Data，点击右边的加号，选中PoseSearchSequenceMetaData, 设置Schema为PSLocomotionSchema，还有就是RootMotion中有个ForceRootLock设置为true，保存。设置如下：
   ![pic](.\PoseSerachRunPic/7.png)

8. 打开ThirdPersonCharacter蓝图，添加CharacterMovmentTrajectory组件，SampleRate设置为10，其他参数不变，设置如下：
   
   ![pic](.\PoseSerachRunPic/8.png)

9.  CharacterMovement设置下行走速度以及摩擦力阻力等，这里设置主要是想让运动跟动画更匹配一些，并且希望停步时有个减速的过程，设置如下：
    ![pic](.\PoseSerachRunPic/9.png)

10. ThirdPersonCharacter的Tick函数编辑如下，需要添加一个TrajectorySampleRange的变量并且是Public，保存：
    ![pic](.\PoseSerachRunPic/10.png)

11. 编辑ThirdPerson_AnimBP动画蓝图，EventGraph和AnimGraph设置如下，MotionMatching Node中的Database设置为PSLocomotionDB：
    ![pic](.\PoseSerachRunPic/11.png)

    ![pic](.\PoseSerachRunPic/12.png)

12. 运行如下
    
    {视频}

13. 如果对动画中的某些帧不满意(比如动捕时的TPose准备帧)，或者不希望在某些帧之间进行过渡，可以通过PoseSerachNotify进行屏蔽
    ![pic](.\PoseSerachRunPic/13.png)
    
## 注意事项
正如开头所说的，这里仅仅能保证项目能够运行起来，很多参数并没有设置，导致效果有点差强人意，我自己挖个坑，希望后面的文章里面能够把所有参数的含义以及解决哪些痛点问题都交代清楚。还有就是上面可能会有些设置不太合理，等我后面深入研究代码后再回来调整吧~

## 下一步
* 接下来可能会写下UE5中如何实现PoseMatching以及PoseSearch代码剖析了
* 在看代码的过程中发现基础不牢靠，比如一些Transform的运算看半天，将来会制作一个学习用的MathLevel~

## 参考资料
* [育碧开源动画库资源](https://github.com/ubisoft/ubisoft-laforge-animation-dataset),为BVH格式，[Github上有导入BVH的脚本](https://github.com/jhoolmans/mayaImporterBVH#installation)，导入BVH后再绑定到模型身上, 可以使用[Motion Matching 中的代码驱动移动和动画驱动移动](https://zhuanlan.zhihu.com/p/432663486)中提到的方法程序化生成根骨骼位置
* 目前来看，PoseSearch跟[《Control》中的Motion Matching](https://zhuanlan.zhihu.com/p/405873194)的方案特别类似，可以作为参考资料