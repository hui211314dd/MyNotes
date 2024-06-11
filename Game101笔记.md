# （3）矩阵变换

## 如何构建旋转矩阵

![RotationMatrix](./Game101Pic/RotationMatrix.png)

因为约定的是列向量形式，因此第一列表示基向量(1, 0)旋转之后的结果，第二列是基向量(0, 1)旋转之后的结果。

## 什么是齐次坐标？

齐次坐标就是将一个原本是n维的向量用一个n+1维向量来表示，简单说就是升维，比如2维的矩阵变成了3维矩阵。

## 为什么需要齐次坐标？

![Translation](./Game101Pic/Translation.png)

如上图所示，如果使用2维的矩阵无法进行平移操作，这对于想通过矩阵的组合实现复杂变换是不方便的。因此通过齐次坐标引入n+1维达到实现平移的目的。

![AffineTransformations](./Game101Pic/AffineTransformations.png)

## 向量或者点如何变成齐次坐标？

![AddCoordinate](./Game101Pic/AddCoordinate.png)

这里的规则是如果是Point那么添加的那一个轴的数值是1，如果是Vector则为0，这样约定有什么好处呢？

* 点如果应用平移操作则会改变点的位置，而向量只有方向性因此平移后点的位置不变。

  ![PointApplyMatrix](./Game101Pic/PointApplyMatrix.png)

  ![VectorApplyMatrix](./Game101Pic/VectorApplyMatrix.png)

* 依然能够满足点和向量的基本性质
  ![Homogenous](./Game101Pic/Homogenous.png)

## 列向量应用矩阵的顺序从右到左

![ComposingOrder](./Game101Pic/ComposingOrder.png)

## 如果围绕一个指定的点旋转？

![DecomposingComplexTransforms](./Game101Pic/DecomposingComplexTransforms.png)

先将指定的点平移到(0, 0)点，然后旋转，最后在平移回原来的位置

## 先线性变换再平移

![LinearTransformFirst](./Game101Pic/LinearTransformFirst.png)

![LinearTransformFirst1](./Game101Pic/LinearTransformFirst1.png)

可以看到先线性变换最后再加上平移变换

# (4) Transformation Cont
