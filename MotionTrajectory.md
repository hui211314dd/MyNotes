UE5中MotionMatching(一) MotionTrajectory

## 前言
我的UE5-Main是在2022.1.4日更新，而且MotionTrajectory和PoseSearch(UE5对MotionMaching的称呼)本身就处于试验阶段，所以不保证将来是否会有大的改动。MotionTrajectory本身代码量不多，想了解UE5 MotionMatching可以从这个插件入手~

MotionTrajectory插件路径:UnrealEngine\Engine\Plugins\Experimental\Animation\MotionTrajectory

PoseSearch插件路径：UnrealEngine\Engine\Plugins\Experimental\Animation\PoseSearch

如果你对Motion Matching感兴趣，可以看下我的其他文章。

[Motion Matching 中的代码驱动移动和动画驱动移动](https://zhuanlan.zhihu.com/p/432663486)

[《荣耀战魂》中的Motion Matching](https://zhuanlan.zhihu.com/p/401890149)

[《最后生还者2》中的Motion Matching](https://zhuanlan.zhihu.com/p/403923793)

[《Control》中的Motion Matching](https://zhuanlan.zhihu.com/p/405873194)

[游戏开发中的Pose Matching](https://zhuanlan.zhihu.com/p/424382326)

[MotionMatching中的DataNormalization](https://zhuanlan.zhihu.com/p/414438466)

## 应用场景
我记得在21年7，8月份看PoseSearch代码的时候，MotionTrajectory插件还没有呢，在9月份新增了这个插件，我们都知道TrajectoryMatching + PoseMatching 是MotionMatching的两大组成部分，Trajectory主要就是应用到MotionMatching功能上，为什么要从PoseSearch中提取出来单独作为一个插件呢？其实MotionTrajectory不仅仅可以用于MotionMatching，也可以作为工具用来提升传统动画的品质，比如帕拉贡的distance matching中需要预测位置点，EA工程师Joakim Hagdahl有一篇博客[predicting-is-guesswork](https://rockhamstercode.tumblr.com/post/175020832938/predicting-is-guesswork)专门讲述Trajectory可以用来改进比如blending in lean poses, Stabilizing facing, Landing anticipation等传统游戏系统品质。Joakim Hagdahl在文章里有一句话让我印象特别深刻
```
These were small incremental improvements but in animation systems every little bit helps to create a more believable motion
```

当然了，UE5中MotionTrajectory主要还是服务于PoseSearch，从提交日志上也可以看出来~

```
Motion Trajectory Component for Motion Matching:

Motion Trajectory Component notes:
1) Abstract component/interface implemented with prediction and history API.
2) Implemented uniform, frame-rate independent history sampling algorithm for retaining trajectory sample coherence.
3) Implemented concrete Character Movement Trajectory Component for encapsulating ground locomotion prediction algorithm and API.
4) Motion Trajectory blueprint library containing:
5) FlattenTrajectory2D algorithm for isolating and removing Z axis direction contribution from tracjectory.
6) ClampTrajectoryDirection for projecting trajectory samples into a discrete, allowed set of directions (such as cardinal).
```

MotionTrajectory究竟提供了啥功能呢？简单说，有以下几项：

* 实现了基础的History数据存储和Future位置的预测(基于CharacterMovement),使用者可以通过继承复写进行自定义
* 实现了两种采样方式分别是基于time和distance
* MotionTrajectoryLibrary提供了若干个处理Trajectory的工具函数，比如Flatten,Rotator,Clamp等

## 跑起来！
在学习一个新东西时，我有一个习惯，我会首先想办法把它运行起来，保证可运行，可调试，然后再看每个功能模块的代码~

1. UE5创建C++的ThirdPerson项目，创建C++的目的主要是将来调试方便~
   
2. Plugins/Animation，启用Motion Trajectory,重启
   ![pic](.\MotionTrajectoryPic/1.png)

3. 编辑ThirdPersonCharacter蓝图，添加CharacterMovementTrajectory组件，并且勾选DebugDrawTrajectory属性为true
   ![pic](.\MotionTrajectoryPic/2.png)

4. 编辑ThirdPersonCharacter Tick方法如下：
   ![pic](.\MotionTrajectoryPic/3.png)

5. 运行

视频效果如下：
{视频 1}

## 代码实现
MotionTrajetory代码特别少，只有三个cpp文件，所以如果想了解UE5的PoseSearch，我认为MotionTrajectory是个很不错的切入点~

![pic](.\MotionTrajectoryPic/4.png)

先看MotionTrajectory.cpp,里面定义了UMotionTrajectoryComponent这个抽象类，作用如下：

* 想要具备History和Preduction功能必须要实现的API,其中包括获取当前Sample世界坐标的CalcWorldSpacePresentTrajectorySample, 获取整个Trajectory的函数GetTrajectory(整个MotionTrajectory最核心的接口)以及根据设置参数获取Trajectory的函数GetTrajectoryWithSettings
  
* 定义好了所需的配置项，比如PredictionSettings表示将来预测Trajectory的设置，HistorySettings表示HistoryTrajectory的设置，SampleRate表示Sample数据是否应该按照指定Rate进行采样(我们知道如果程序Tick或者Timer采样的话，采样数据跟Tick或者Timer的更新频率有关，但如果需要指定Rate的话，这个参数就有用了，比如Tick按照1s一帧，5s过后存储的数据分别是-5s，-4s，-3s，-2s，-1s的数据，但是PoseSearch可能想要的是-(1/30)s，-(2/30)s，-(3/30)s...的数据，设置SampleRate = 30并且bUniformSampledHistory为true的话，可以在-(1/30)s，-(2/30)s，-(3/30)s...通过插值计算拿到数据,更多细节可以看GetHistory函数)，MaxSamples定义了存储History数据的环形数据结构的初始大小，bSmoothInterpolation定义了GetHistory函数中的插值算法是否采用[Centripetal Catmull–Rom Spline Interpolation](https://en.wikipedia.org/wiki/Centripetal_Catmull%E2%80%93Rom_spline),曲线拟合慢但更准确，线性插值快但误差相对较大，需要自己权衡
  
* FMotionTrajectorySettings中的Domain可以设置采样方式，基于Time或者Distance或者两者混合,需要注意的是，如果指定的是Time，内部仍然会存储和计算distance数据，这里指定的time表示采样范围基于Time,比如设置的是2s,那么3s的数据就不再存储了

* TickHistoryEvictionPolicy函数通过Tick或者Timer被调用，会遍历所有HistorySample并设置内部Offset,当发现该Sample已经不在范围内时，立即对该Sample进行移除

![pic](.\MotionTrajectoryPic/5.png)

### UMotionTrajectoryComponent总结
UMotionTrajectoryComponent提供了基本的存储结构以及配置项，通过插值生成满足SampleRate要求的HistorySample以及
HistorySample的更新剔除；可以发现这个类里面并没有生成预测点，因为预测点是由UCharacterMovementTrajectoryComponent生成的

UCharacterMovementTrajectoryComponent继承UMotionTrajectoryComponent，重写了CalcWorldSpacePresentTrajectorySample，GetTrajectory以及GetTrajectoryWithSettings函数，GetTrajectoryWithSettings调用PredictTrajectory生成了预测数据，所以UCharacterMovementTrajectoryComponent的核心就是PredictTrajectory，如果你对CharacterMovement正常walk有了解的话，看这里的代码会很容易，这里主要就是参考了CalcVelocity和PhysicsRotation的函数实现，CalcVelocity用来模拟生成速度矢量，PhysicsRotation用来模拟生成角色朝向，PredictTrajectory的核心就是利用CharacterMovement的运动模型去预测将来n秒后人物的位置朝向速度等信息。[知乎里分享CharacterMovement的文章太多了](https://zhuanlan.zhihu.com/p/34257208)，就不多解释了

### UCharacterMovementTrajectoryComponent总结
继承UMotionTrajectoryComponent，根据CharacterMovement的运动原理增加了未来运动的预测，生成预测的FutureTrajectory数据,当用户调用GetTrajectory时，将HistoryTrajectory和FutureTrajectory组装返回

UMotionTrajectoryBlueprintLibrary比较简单，提供了对Trajectory后期处理的若干函数，包括Flatten压平，旋转，ClampTrajectoryDirection等，需要注意的是Flatten并不是单单移除Z这么简单，有个可选项是否保留速度，如果保留速度的话，说明要保留速度值，Distance需要重新计算。下面的视频是Flatten后的效果：

{视频}

## 注意事项

* UCharacterMovementTrajectoryComponent的命名容易让人疑惑，以为是重载了CharacterMovement，其实不是，UCharacterMovementTrajectoryComponent继承于ActorComponent，不要混淆了

* MotionTrajectory提供的功能特别简单基础，目前的代码能够跑通MotionMatching的流程以及支持Time和Distance Matching，框架定了，大部分的功能代码还需要自己写

* 有些表现不是很理解，感觉有bug

* 如果看过上面[Motion Matching 中的代码驱动移动和动画驱动移动](https://zhuanlan.zhihu.com/p/432663486)和[《荣耀战魂》中的Motion Matching](https://zhuanlan.zhihu.com/p/401890149)的文章，里面都提到了结合碰撞检测生成Trajectory，比如前方是墙壁或者拐角的时候会影响FutureTrajectory，MotionTrajectory并没有做任何操作，需要自己根据项目需要自己添加

* UCharacterMovementTrajectoryComponent某些函数在MOVE_Walking状态下才有意义，可以猜测，UE5的PoseSearch的主要用途也是地面的locomotion

## 相关的参考资料
* EA工程师Joakim Hagdahl的这篇文章[predicting-is-guesswork](https://rockhamstercode.tumblr.com/post/175020832938/predicting-is-guesswork)利用Trajectory改进传统动画系统特别有用，不过大部分功能我没有碰到过，所以等需要的时候再参考看吧
  
* [《Control》中的Motion Matching](https://zhuanlan.zhihu.com/p/405873194)中提到了Distance Based Matching，应用的话可以作为参考

* 角色运动模型除了CharacterMovement这种之外，其实还有很多种方式，比如《荣耀战魂》采用的是基于Spring Damper构建的Controller，这里推荐育碧大神Daniel Holden的这篇文章[Spring-It-On: The Game Developer's Spring-Roll-Call](https://theorangeduck.com/page/spring-roll-call)，Daniel Holden的论文[Learned Motion Matching](https://theorangeduck.com/page/learned-motion-matching)的运动模型就是基于此
* 
{视频2}

## 我的疑问
* EffectiveTimeDomain干啥用的，有知道的朋友记得告诉我~
* 如果文章有错误，记得联系我~