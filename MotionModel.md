起步/稳定时：  
* MaxSpeed需要根据AnalogInputModifier系数进行调整(与MinAnalogSpeed比较取最小)
* (加速度不为0的前提下)apply 摩擦力的影响
```C
// Friction affects our ability to change direction. This is only done for input acceleration, not path following.
const FVector AccelDir = Acceleration.GetSafeNormal();
const float VelSize = Velocity.Size();
Velocity = Velocity - (Velocity - AccelDir * VelSize) * FMath::Min(DeltaTime * Friction, 1.f);
```
* (加速度不为0的前提下)apply 加速度的影响
```C
const float NewMaxInputSpeed = IsExceedingMaxSpeed(MaxInputSpeed) ? Velocity.Size() : MaxInputSpeed;
Velocity += Acceleration * DeltaTime;
Velocity = Velocity.GetClampedToMaxSize(NewMaxInputSpeed);
```

停步时：  
```C
const float ActualBrakingFriction = (bUseSeparateBrakingFriction ? BrakingFriction : Friction);
ApplyVelocityBraking(DeltaTime, ActualBrakingFriction, BrakingDeceleration);

// 关于ApplyVelocityBraking，如果摩擦力为0则无需迭代，直接通过Velocity = Velocity + (RevAccel) * dt计算即可;如果有摩擦力，需要迭代计算，公式等于Velocity = Velocity + ((-Friction) * Velocity + RevAccel) * dt; 记得需要判断如果计算后的速度跟OldVelocity方向相反直接设置为ZeroVector即可。
```
















CharacterMovementComponent参数详细解释：  
Gravity Scale： 重力系数   
Max Acceleration：最大加速度，真正应用的加速度取决于你摇杆的程度(InputVector)  
Braking Friction Factor: 停步摩擦力系数，历史原因默认值为2，应用场景为停步时或者当前速度超过MaxSpeed时(简称Braking)，摩擦力Friction需要乘以这个系数  
Braking Friction, Use Separate Braking Friction, Ground Friction:在Walk Movement Mode场景下，如果勾选了Use Separate Braking Friction，在Braking时，使用Braking Friction,否则使用Ground Friction,如果不是Braking的情况，正常起步或者走路，使用的是Ground Friction  
Max Walk Speed: Walk最大速度，超过时需要Brake  
Braking Deceleration Walking: Braking时的阻力，应用时取当前速度反方向的单位向量乘以这个阻力值，得到阻力的加速度。  
Braking Sub Step Time: 存在摩擦力的情况下，计算Braking时需要分时间片计算提高计算精准度，但是时间片分的太小的话计算量大会影响性能，这个时间片的时间长度用来控制平衡。  



