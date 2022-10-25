# 《最后生还者2》中的MotionMatching(工具篇)

## 前言

为什么写这篇呢？ 两个原因，一是之前写过一篇[《最后生还者2》中的Motion Matching](https://zhuanlan.zhihu.com/p/403923793), 但那个时候没有听懂MMAssistant和LoopsInsertTool的讲解，所以在当时就挖了个坑; 二是写过[UE5中的MotionMatching-实践(一)](https://zhuanlan.zhihu.com/p/557039076),[一键生成动画-育碧蒙特利尔关于Mocap的自动化技术(上)](https://zhuanlan.zhihu.com/p/568048965)和[一键生成动画-育碧蒙特利尔关于Mocap的自动化技术(下)](https://zhuanlan.zhihu.com/p/569672931)后，想进一步看下其他公司如何高效编辑动捕动画，从我找到的资料中看除了顽皮狗以外，其他公司都没有重点介绍这块，但我在之前的文章中也提过，如果要达到产品级的效果，如何高效编辑动捕动画这个大坑必须得迈过去。基于这两个原因我硬着头皮再把顽皮狗的那次演讲又看了一遍，这次感觉收获颇丰，看了之后啊觉得常看常新...

![常看常新](.\TLOU2MotionMatchingToolsPic/1.png)

## MMAssistant

{MMAssistant演示视频.mp4}

MMAssistant是Maksym Zhuravlov提供给动画师的一个辅助工具，协助动画师制作用于MotionMatching所需的动画，该辅助工具人性化地设定了多个步骤，不管你是新手动画师还是经验丰富的动画师，只要按照辅助工具提供的步骤一步一步来，就能产出高质量可用的动画。

接下来我们看下MMAssistant的工作流程是怎么样的。

### Drop In Mocap Data And Finding the best take
插入多条动捕动画，在其中比对寻找最佳的动画片段，比如第一个例子制作的是RunForwardStart的动画，我们从两者中找到了一个理想的动画。

![在所有的动捕动画中选取最合适的动画](.\TLOU2MotionMatchingToolsPic/2.png)

### Select Your Loco
这时候我们再选择下我们要编辑哪个人物哪种状态的什么类型的动画，MotionMatching具体需要多少种动画呢？没有人比动画TA更了解了，所以具体需要什么动画需要动画TA以yaml文件的形式填充，动画师只需要把所有的动画补齐就可以了。

可以看到主界面上方需要选择人物比如艾莉或者艾比，然后选择人物的状态比如relaxed，uneasy，explore，battle等，最后需要选择动画的类型比如walk, jog, run等等, 当选择完毕后下方会出现该人物在这个状态下所需的所有动画类别，下面列出的分别是：

* ellie-mm-explore-run-loops，所有loops相关的动画
* ellie-mm-explore-run-starts, 所有starts起步相关的动画
* ellie-mm-explore-run-stops, 所有stops停步相关的动画
* ellie-mm-explore-run-quickplants, 所有quickplants相关的动画 
* ellie-mm-explore-run-corners, 所有corners拐角相关的动画
* ellie-mm-explore-run-turns-on-spot, 所有原地转身的动画

点击ellie-mm-explore-run-starts后弹出了右侧所有待补充的starts动画列表:

![MMAssistant主界面](.\TLOU2MotionMatchingToolsPic/3.png)






### Path Correction Step
MMAssistant工具在这一步干了三件事情：
* 生成了角色在该动画下的DesiredTrajectory, 用绿线表示，该DesiredTrajectory参考了角色MotionModel的一系列参数包括SpringParam, DPS等等
* 同时也生成了CharacterTrajectory, 该Trajectory反映了动画本身的运动路线, 用红线表示
* 在场景中生成了path-corrector node,通过旋转平移可以将CharacterTrajectory尽可能地对齐到DesiredTrajectory

可以清楚的看见当调整path-corrector时整个动画也会跟着偏移。通过路线校正，不仅可以将动画路线尽可能地对齐到直线上(避免出现较大的空隙)，也可以有效地避免新手动画师将动画放到错误的配置中。

![PathCorrection](.\TLOU2MotionMatchingToolsPic/4.png)

调整完毕后，继续点击左侧的‘OK I'm DONE!’, 进入下一个步骤Root Motion Correction

### Root Motion Correction
MMAssistant工具在这一步会根据角色重心位置(CenterOfMass)生成近似的RootMotionCurves, 大概每10-15frames，名字为AnimAlignNode, 动画师需要在这里修改的项目有Curves, lateral motion, tangents以及Y-rot等，修改的目标是保证曲线平滑以及很好地涵盖到运动的内容上。

![RootMotionCorrection](.\TLOU2MotionMatchingToolsPic/5.png)

通过视频可以看到修改AnimAlignNode并没有影响现有的动画位置，两者独立的好处是Align即满足了MotionMatching查询的需要，又没有破坏原有动画的节奏。

在这里动画师需要小心翼翼的工作，目的就是“smooth and covers the motion well”，前一个目标是让整个曲线(包括数值和切线)平滑，后面不仅要求Align的位置能够与角色尽可能地贴合，还要求Y-rot能够跟角色贴合，并且Y-rot还有DPS的限制要求，我们知道在角色MotionModel中定义了8方向旋转速度DPS, 如果发现动画的DPS与MotionModel定义的DPS相差太大的话，需要调整动画内容。

![左侧显示的是玩家角色动画的DPS右侧显示的是某个NPC角色的MotionModel DPS设置](.\TLOU2MotionMatchingToolsPic/7.png)


调整完毕后，继续点击左侧的‘OK I'm DONE!’, 进入下一个步骤Set Engine FrameRange

### Set Engine FrameRange
在第一步的时候我们设置了粗粒度的Range，而这一步为了导出到引擎需要设置更为准确的Range。在设置StartFrame需要注意的一点就是“No Dead Frames”，设置EndFrame时有两条需要注意:
* EndFrame那一帧要有意义，比如循环帧的开始或者其他核心Pose等
* 找好理想帧后不要立即设置为EndFrame，需要计算EndFrame + Future MotionModelTrajectory, 将这一个时间设置为EndFrame，为什么这样呢？因为顽皮狗的MotionMatching在SearchFrame时，如果该帧没有指定时间的FutureTrajectory则直接抛弃(举例来说，设置的FutureTrajectory最远为1s或者说30frames, 如果一个Stop动画长度为40frames, 那么MotionMatching只会考察[1, 10], 后面的不会考虑).

### Finish!
设置好StartFrame以及EndFrame，再点击‘OK I'm DONE!’，工具提示我们工作已经完成了，可以继续点击‘ALL DONE! Process that clip!’,点击后该工具会在后台帮我们把指定Range的动画切成独立的.ma文件放到正确的文件夹里并将相关数据以meta-data的形式导出，并且完成ControlRig的绑定。

完成这个以后动画师可以马不停蹄地编辑下一个了！

![已经完成的标记](.\TLOU2MotionMatchingToolsPic/6.png)

整个视频分别演示了各种Start, Corners, Strafe以及Stop动画等, 很具参考意义。

## LoopsInsertTool

{LoopsInsertTool视频.mp4}

LoopsInsertTool是Maksym Zhuravlov提供给动画师的另一个辅助工具，

## Animation Analyzer

## DebugTools

## 我的总结
Align与动画分离

调整Trajectory的作用

成本，动画的自然与节奏

高效的工作流，TA的作用

DPS的处理

## Bonus!

## Next