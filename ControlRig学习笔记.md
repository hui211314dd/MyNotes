## 执行流程
Content Browser打开ControlRig资源后会执行Setup以及ForwardsSolve

点击Compile同样是先执行Setup再执行ForwardsSolve

点击SetupEvent会只执行SetupEvent，不会执行ForwardsSolve流程

点击ForwardsSolve或者移动Control时仅仅执行ForwardsSolve流程

同样的ControlRig导入不同模型时，骨骼的InitialTransform始终不变，但是GlobalTransform，LocalTransform会发生改变，如何保证不同模型下行为一致的呢？ControlRig如何完成类似于Retarget的操作的呢？

## GetTransform

当前Initial的位置等于InitialTransform + OffsetTransform

当前Current的位置等于CurrentTransform + OffsetTransform

## SetControlOffset

## SetTransform
Initial为true设置InitialTransform否则设置为CurrentTransform(真正设置的值要考虑OffsetTransform ？ 与OffsetTransform区别？)

## OffsetTransform


## Bone/Control/Space



## ProjectToNewParent

