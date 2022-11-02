我们以MotionBuilder中的Mia举例

## 创建MiaTPose的fbx
将mia_rigged拖入场景，No Animation，可以看到Y轴朝上，Mia面向Z轴方向，保存，命名为mia_tpose.fbx

## 载入动画文件
将mia_fk_runstopturn拖入场景，fbx Open/Take001, 默认情况下已经定义好ControlRig了

## PathCorrection
动画并没有在标准轴上运动，并且初始帧不一定在原点位置，而且初始帧朝向可能并没有指向Z轴，PathCorrection就是要处理这些整理工作，做了这些以后，后面调整曲线的步骤就会好做很多。

![1](.\MotionBuilderRootMotionPic/0.png)

* 打开Stroy后新建CharacterTrack，将Character设置为Mia
* 在轨道上右键，选择Insert Current Take
* 选中轨道中的take, 点击Tack中的小眼睛图表(如果置灰说明Story没有开启)，打开Ghost显示，并且可以移动旋转整个动画，这时候可以对齐位置, 你可以使用Clip Ghost或者手动选中Hips或者把Trajectory打开，辅助进行旋转
* 对齐完毕后右键轨道选择Plot Whole Scene to Current take.

## 定义Relation约束
`Ctrl + W`进入Node模式，选中Mia:Root节点，将Relation拖到Mia:Root节点上，选择Constraints, 接下来就可以在Constraint Setting中编辑如何约束了

![1](.\MotionBuilderRootMotionPic/1.png)

设置完毕后，我们查看Mia:Root的属性，发现已经有值了。

## 将动画Bake到ControlRig上
Bake到ControlRig后，Relation就可以去掉Active了。

## 编辑Mia:Root上的曲线
这时候运动数据还没有Bake到骨骼上，所以这时可以编辑Mia:Root上的属性信息，比如平滑，对齐等等

![Smooth](.\MotionBuilderRootMotionPic/2.png)

我们再看下X轴的曲线，我们本来以为mia_fk_runstopturn是沿着Z轴方向跑的，结果发现X轴方面也有偏移，这也是《最后生还者2》中的MotionMatching(工具篇)提到的PathCorrection的作用，否则动画师在编辑X轴时比较困难。并且PathCorrection应该在Relation约束之前操作。

![如果没有PathCorrection则X轴，Z轴编辑起来都很麻烦](.\MotionBuilderRootMotionPic/3.png)

最后再平滑旋转信息

![平滑旋转信息](.\MotionBuilderRootMotionPic/4.png)

## Bake到骨骼上

调整完毕后我们Bake到骨骼上。

然后保存为mia_runstopturn_rootmotion.fbx

## 导入到虚幻引擎中

先导入mia_tpose.fbx，然后再导入mia_runstopturn_rootmotion.fbx

打开mia_runstopturn_rootmotion，EnableRootMotion设置为true，就可以使用该动画了。