# Lyra中的相机系统

## 前言

本篇会先讲UE5相机系统流程以及TPS中SpringArmComponent的实现，然后说下Lyra中如何实现数据驱动的相机系统，最后总结下LyraCameraSystem实现的优势和目前的不足。

## 引擎默认的相机系统

### CameraSystem流程图
![UnrealCameraSystem](./LyraCameraPic/UnrealCameraSystem.png)

CameraSystem有几个要点：
* 输入改变ControlRotation;SpringArmComponent使用ControlRotation改变ChildActor(即CameraComponent)的RelativeTransform;CameraManager在计算最终相机POV时最终调用的是Pawn身上挂载的CameraComponent的GetCameraView函数

* SpringArmComponent的核心在于UpdateDesiredArmLocation函数，这个函数会计算相机最终的位置和朝向，如果设置了bUsePawnControlRotation那么会使用ControlRotation, 而最终位置就需要使用CollisionTest的函数，比如SweepSingleByChannel，如果发生了碰撞，那么最终的位置跟碰撞点有关，如下图所示：
  
  ![CollisionTest](./LyraCameraPic/SpringArmComponent.png)

* PlayerCameraManager中有个很重要的函数是UpdateViewTarget，只要重载这个函数就可以接管Gameplay中Camera的管理。

* CameraComponent中的重要函数是GetCameraView，正如上面所提到的计算最终的POV时调用的函数正是CameraComponent::GetCameraView, 因此重载这个函数可以做很多Gameplay相关的逻辑或者设计，Lyra正是利用了这一点。

## Lyra相机系统

我们知道Lyra的设计理念是数据驱动和ModularPlay，不管是玩法，技能还是UI, 当然还有Camera，我们先通过流程图看下配置的CameraMode是如何参与进来的，然后再说下CameraMode以及相关的代码解释。

### 流程图

(PawnData中配置以及如何生效)

### LyraCameraStack

### LyraCameraMode

#### LyraCameraMode_ThirdPerson

#### LyraCameraMode_TopDownArenaCamera

### LyraPenetrationAvoidanceFeeler

## Lyra示例

### 过肩视角

### 开镜实现

### 射击时目标判断

(AimUI以及碰撞判定逻辑)

## Lyra对比传统方案的优势以及不足

数据驱动的优势
Feeler对比SpringArm的优势

## 总结

(项目中可以借鉴的地方)
1. Feeler方式
2. CameraStack和CameraMode模式，DataDriven，有数据驱动的优势，可配置在Experiences中
3. 有些可惜代码在开发中，还有很多需要填补的