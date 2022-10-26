# 《最后生还者2》中的MotionMatching(工具篇)

## 前言

为什么写这篇呢？ 两个原因，一是之前写过一篇[《最后生还者2》中的Motion Matching](https://zhuanlan.zhihu.com/p/403923793), 但那个时候没有听懂MMAssistant和LoopsInsertTool的讲解，所以在当时就挖了个坑; 二是写过[UE5中的MotionMatching-实践(一)](https://zhuanlan.zhihu.com/p/557039076),[一键生成动画-育碧蒙特利尔关于Mocap的自动化技术(上)](https://zhuanlan.zhihu.com/p/568048965)和[一键生成动画-育碧蒙特利尔关于Mocap的自动化技术(下)](https://zhuanlan.zhihu.com/p/569672931)后，想进一步看下其他公司如何高效编辑动捕动画，从我找到的资料中看除了顽皮狗以外，其他公司都没有重点介绍这块，但我在之前的文章中也提过，如果要达到产品级的效果，如何高效编辑动捕动画这个大坑必须得迈过去。基于这两个原因我硬着头皮再把顽皮狗的那次演讲又看了一遍，这次感觉收获颇丰，看了之后啊觉得常看常新...

![常看常新](.\TLOU2MotionMatchingToolsPic/1.png)

这个期间特别感谢Maksym Zhuravlov的耐心解答，耐心地说了很多有用的细节，帮助特别大，好人一生平安!

## MMAssistant

{MMAssistant演示视频.mp4}

MMAssistant是Maksym Zhuravlov提供给动画师的一个辅助工具，协助动画师制作用于MotionMatching所需的动画，该辅助工具人性化地设定了多个步骤，不管你是新手动画师还是经验丰富的动画师，只要按照辅助工具提供的步骤一步一步来，就能产出高质量可用的动画。

接下来我们看下MMAssistant的工作流程是怎么样的。

### Drop In Mocap Data And Finding the best take
拖入多条动捕动画，在其中比对寻找最佳的动画片段，比如第一个例子制作的是RunForwardStart的动画，我们从两者中找到了一个理想的动画区间。

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

### Get Rough Time Range
正如前面提到的，我们打算制作RunForwardStart的起步动画，所以选择的是右侧的ellie-mm-explore-idle^run-strafe-fw-l-foot, 随后左边就是出现一个新的按钮提示选择StartFrame,在上面Finding the best take时已经找到了理想的动画区间，所以将时间轴拖到开始位置时点击‘Select START Frame’即可，随后又会出现‘Select END Frame’,同样将时间轴拖到相应位置点击即可 (这里注意一点的是可以将范围设置的略宽一些，后面会专门对导出范围进行细调)。随后又出现一个新的按钮‘OK,I'm DONE’, 点击该按钮，则要进行下面的运动路线校正。

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

LoopsInsertTool是Maksym Zhuravlov提供给动画师的另一个辅助工具，辅助动画师处理循环动画和过渡动画接缝处之间的差异。以Start动画举例，我们希望Start动画后能够接上Loop动画，但是Start动画末尾帧与Loop动画相应的动画帧差异太大可能会导致Search不到，LoopsInsertTool工具可以帮助修复这种差异。

![本质是个动画数据库](.\TLOU2MotionMatchingToolsPic/8.png)

LoopsInsertTool存储了各个角度Loops动画的曲线数据，可以很容易地访问(用于PoseMatch对比)以及部署到Maya的动画层中。我们下面看下它的工作流是怎样的。

### 找到合适的匹配帧
首先是需要动画师确认下希望在哪一帧开始与Loop动画融合，比如视频中是第54帧

### 插入Loop动画到新的Layer中
将时间轴切到54帧后，点击LoopsInsertTool相应的Loop动画，比如当前编辑的是RunForwardStart动画，将来要融合的动画应该是run-loop-fw, 所以点击ellie-mm-explore-run-loop-fw即可，点击后工具主要做了两件事情:

* 插入Loop动画到新的Layer中
* LoopsInsertTool工具出现了新的面板，可以调整当前Pose的位置比如X, Z, XZ, ZX以及调整Phase等

![插入Loop动画](.\TLOU2MotionMatchingToolsPic/9.png)

动画师需要在这里调整Phase, X，Z等以至于当前的Loop动画帧与Start的动画帧动作大致对齐上

![通过调整参数让Loop帧匹配上](.\TLOU2MotionMatchingToolsPic/10.png)

位置和Pose对齐好后，点击‘Select Layer Weight Curve’按钮，开始调整Layer的权重。

### 调整Layer权重
在这一步需要调整BaseLayer与NewLayer的权重关系，比如平滑程度，平滑时间等。这一步特别重要，你可以不用导入到游戏引擎而在Maya中就可以看到Start切换到Loops时的效果，这可以帮助动画师快速解决接缝处不连续的问题。

![调整Layer权重](.\TLOU2MotionMatchingToolsPic/11.png)

调整完毕后点击左下角的‘All done!’, 下一步调整动画的AlignNode曲线

### 调整Align
工具会将Loop动画对应的Align插入当前动画的AlignNode曲线中，动画师可以在这里进行调整

### 手动粘贴标准Idle
新建Layer命名为idle_pose, 在动画开头粘贴上IdlePose, 这对于MotionMatching Search特别有用

### KeyFrame
手动打磨动画细节等

## Animation Analyzer
{Animation Analyzer视频演示.mp4}

Michal Mach介绍了他写的工具‘Animation Analyzer’,它可以程序化检查指定动画，根据MotionModel的要求，检查出哪些片段不规范，黄色的话是Warning，红色的话是Error，通过一些可视化的界面(比如DPS超过了阈值，Align速度曲线可视化，可以查一些速度突变的问题等)辅助动画师

视频演示共解决了两个常见的问题，一是Align旋转的DPS超过了MotionModel中约定的值，动画DPS为284，而MotionModel定义的为180，超了50%，所以给出了Error提示。打开该动画，右上角有一个修改DPS的小工具

![修改DPS小工具](.\TLOU2MotionMatchingToolsPic/12.png)

点击GetDPS可以计算当前动画的DPS值，可以看到出现了“DPSis284.2[26-45] Add 11 frames”,表示需要添加11frames，DPS才会变成180，右上角三个棕色按钮分别表示向左侧添加Frames, 向两侧添加Frames以及向右添加Frames; 下面两个黄色按钮可以向左或向右整体移动DPS涉及到的Frames。修改后再次检测，发现DPS的问题已经解决了。

另外一个问题视乎是速度变化异常，可能跟曲线的切线异常有关

![工具提示视乎是bad tangent](.\TLOU2MotionMatchingToolsPic/13.png)

打开可视化Tips发现确实切线异常

![速度变化异常](.\TLOU2MotionMatchingToolsPic/14.png)

发现原因后修改就很简单了

![速度变化异常](.\TLOU2MotionMatchingToolsPic/15.png)

## 我的总结
Align的编辑，可以理解以meta-data的方式

PathCorrection的作用, 要知道演员在动捕走直线时，走的路线也可能存在偏移的，比如偏了10度左右，通过PathCorrection可以将路线对准到标准线上，这样生成的RootMotion也都是很容易编辑的值(都在标准轴上)，但是如果偏差太大，PathCorrection旋转角色后，起始帧角色的朝向不是错了吗？我猜可以通过约束或者类似于AdjustmentBlend的技术解决

如果你仔细看，可以发现MMAssistant和LoopsInsertTool都没有提到如何让Animation匹配MotionModel的参数，MotionMatching一直强调数据一致性，但是PlayerAnimSet在编辑时并没有提到这些，这是为什么呢？主要原因有两点：一是成本，二会破坏动画的自然与节奏，Maksym Zhuravlov在其他方面做了努力，包括PlayerWeightStrategy, IK, Camera等等，特别是过肩的Camera，可以让玩家更关注动画整体的流畅表现而不纠结在脚上。NPC的策略则不同，NPC的策略是MotionModel尽可能去适配Animation, 加上Clamp, NPC可以有效解决滑步的问题

高效的工作流，这是显而易见的，这也是顽皮狗反复强调资深动画TA的原因

游戏角色的DPS是动画驱动的，配合每0.3-0.5s的实时Correction, 这也是为什么《最后生还者2》角色旋转时角色面对镜头是延迟的。