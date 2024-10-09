# 适用于单人生存的单区块高效南瓜种植解决方案

## 目录

[随机刻](###随机刻)

[南瓜的生长原理](###南瓜的生长原理)

[高效种植方案](###高效种植方案)

[高效种植方案的应用设计](###高效种植方案的应用设计)

[瓜机方案的优缺点](###瓜机方案的优缺点)

[参考文献](###参考文献)

## 前言

通过本文，你将了解南瓜的生长原理，高效种植方案，根据高效的种植方案设计的南瓜机，以及该方案的优缺点。由于西瓜和南瓜它们的生长原理一样，本文主要以南瓜为例。

## 正文

南瓜理论产量的公式推导比较硬核，看不懂的可以直接跳转至[结论](###结论)

### 随机刻

为了计算出南瓜的理论产量，我们需要了解一些有关随机刻的知识，该部分会用到概率论的相关知识

打开F3+G，游戏进入调试模式，水平方向上每个16*16的格子为一个区块，每个区块中沿垂直方向上没16个为一个子区块，即为一个边长为16的正方体，在每游戏刻中每个子区块中随机选择的方块，被选中的这些方块会被给予一个“随机刻”。每一节被抽选的方块个数可右指令`/gamerule randomTickSpeed` 查看和更改。 randomTickSpeed 的默认值为3。也就是说在每游戏刻中每个子区块中会独立重复且随机的选择三个方块执行随机刻，注意可以同时选中一个方块。如果成熟的南瓜梗被给予随机刻，如果周围环境合适，就会生长出南瓜。

显然在 randomTickSpeed = 3 的情况下，在一个子区块（16\*16\*16共计4096个方块）中，每游戏刻选中目标方块可以记为事件A，该事件服从二项分布 $A\sim b(3, \frac{1}{4096})$ ，根据二项分布的概率公式，容易得到每游戏刻内至少选中一次目标方块的概率 
$$
P\left \{ A≠0 \right \}  = 1-C_{3}^{0} (\frac{1}{4096} )^{0}(1-\frac{1}{4096})^{3} = 0.0732\%
$$
设在t游戏刻内，目标方块被随机刻选中的次数（如果在每个游戏刻选中选中目标方块多次按照一次来看），记为事件T，事件T服从二项分布 $T\sim b(t,P\{A\ne 0\})$ 。那么在t游戏刻内，某方块被选中的概率
$$
P\left \{ T≠0 \right \}  = 1-C_{t}^{0} (P\{A\ne 0\})^{0}(P\{A\ne 0\})^{t} = 1-(1-0.0732\%)^{t}
$$
根据以上公式可以验算一下

一秒（20gt）内被选中至少一次：
$$
P\left \{ T≠0 \right \} |_{t=20} = 1-C_{20}^{0} (0.0732\%)^{0}(1-0.0732\%)^{20}=1.45\%
$$
五分钟内（6000gt）被选中至少一次：
$$
P\left \{ T≠0 \right \} |_{t=6000} = 1-C_{6000}^{0} (0.0732\%)^{0}(1-0.0732\%)^{6000}=98.76\%
$$
946gt被选中至少一次：
$$
P\left \{ T≠0 \right \} |_{t=946} = 1-C_{946}^{0} (0.0732\%)^{0}(1-0.0732\%)^{946}=49.98\%
$$
这里和wiki[^1]的数据基本一致，如图所示。

![2022-10-19_093848](./pic/1.png)

根据二项分布的期望公式，我们可以得到在t游戏刻内，选中目标方块的次数：
$$
E(T) = t*P\{A\ne 0\} = 0.0732\%t
$$
因此我们可以得到一小时（72000游戏刻）内选中目标方块的次数为：
$$
E(T)|_{t=72000} = 0.0732\%\times 72000 = 52.7/h
$$
在计算南瓜的理论产量时我们会用到上述公式。		

### 南瓜的生长原理

南瓜的生长原理相对简单许多，下面是截取1.17源码中梗类方块的randomTick逻辑代码[^2]：

```java
 public void randomTick(BlockState blockstate, ServerLevel level, BlockPos blockpos, Random random) {
        if (level.getRawBrightness(blockpos, 0) >= 9) {
            //获得生长速度
            float f = CropBlock.getGrowthSpeed(this, level, blockpos);
            //根据生长速度，概率判断是否生长
            if (random.nextInt((int) (25.0F / f) + 1) == 0) {
                int i = blockstate.getValue(AGE);
                if (i < 7) {//南瓜梗没有成熟
                    blockstate = blockstate.setValue(AGE, Integer.valueOf(i + 1));
                    level.setBlock(blockpos, blockstate, 2);
                } else {//南瓜梗成熟
                    Direction direction = Direction.Plane.HORIZONTAL.getRandomDirection(random);//随机获得一个方向
                    BlockPos blockpos = blockpos.relative(direction);//获得随机方向相邻的方块坐标
                    BlockState blockstate = level.getBlockState(blockpos.below());//获得随机方块的下方方块的状态信息
                    //判断随机方向的方块是否为空气且其下方是否为耕地或泥土类方块
                    if (level.getBlockState(blockpos).isAir() 
                        && (blockstate.is(Blocks.FARMLAND) 
                            || blockstate.is(BlockTags.DIRT))) {
                        level.setBlockAndUpdate(blockpos, this.fruit.defaultBlockState());
                        level.setBlockAndUpdate(blockpos,
                                this.fruit.getAttachedStem().defaultBlockState()
                                                .setValue(HorizontalDirectionalBlock.FACING, direction));
                    }
                }
            }
        }
    }
```

下面是获得生长速度getGrowthSpeed的逻辑代码：

```java
protected static float getGrowthSpeed(Block block, BlockGetter blockgetter, BlockPos _blockpos) {
      float f = 1.0F;
      BlockPos blockpos = _blockpos.below();
    //判断耕地状态
    for(int i = -1; i <= 1; ++i) {
         for(int j = -1; j <= 1; ++j) {
            float f1 = 0.0F;
            BlockState blockstate = blockgetter.getBlockState(blockpos.offset(i, 0, j));
            if (blockstate.is(Blocks.FARMLAND)) {
               f1 = 1.0F;
               if (blockstate.getValue(FarmBlock.MOISTURE) > 0) {f1 = 3.0F;}
            }
            if (i != 0 || j != 0) { f1 /= 4.0F; }
            f += f1;
         }
      }
      //判断附近是否有南瓜梗
      BlockPos blockpos1 = _blockpos.north();
      BlockPos blockpos2 = _blockpos.south();
      BlockPos blockpos3 = _blockpos.west();
      BlockPos blockpos4 = _blockpos.east();

      boolean flag = blockgetter.getBlockState(blockpos3).is(block) || blockgetter.getBlockState(blockpos4).is(block);
      boolean flag1 = blockgetter.getBlockState(blockpos1).is(block) || blockgetter.getBlockState(blockpos2).is(block);
      //毗邻只有一个方向（比如南北向，或者东西向）有南瓜梗，则对生长速度没有影响
      //如果两个方向均有南瓜梗，则f减半
      if (flag && flag1) {f /= 2.0F;} else {
        //如果对角毗邻方块有南瓜梗，则生长速度减半
         boolean flag2 = blockgetter.getBlockState(blockpos3.north()).is(block) 
             || blockgetter.getBlockState(blockpos4.north()).is(block)
             || blockgetter.getBlockState(blockpos4.south()).is(block)
             || blockgetter.getBlockState(blockpos3.south()).is(block);
         if (flag2) {f /= 2.0F;}
      }
      return f;
   }
```



将以上代码解读为以下几点：

当南瓜梗被选中随机刻后，

1. 首先判断南瓜梗处的光照是否≥9，如果<9，则南瓜不会生成，可以利用这个原理来实现瓜机的开关。

2. 接着计算生长速度，记为Sg（Growth Speed），Sg初始值为1。其具体计算过程如下：

   1. 判断梗周围的耕地数目

      在以梗为中心的3*3的区域内，梗附着的耕地如果为干耕地Sg加1，湿耕地Sg加3，其毗邻的8个方块每块干耕地加1/4，每块湿耕地加3/4。

   2. 判断梗周围相同梗

      如果南瓜梗毗邻只有一个方向（比如只有南北向，或者只有东西向）有南瓜梗，则对生长速度没有影响，如果两个方向都有南瓜梗或者斜角方向至少有一个南瓜梗，生长速度Sg直接减半，我们可以认为这是过密种植了。是否过密种植只是由一个判断来决定，因此种植南瓜时最好不要触发上面的过密密植条件。

      对于梗的判断可能比较容易误解，下面将展示具体情况：

      1. 不会触发过密密植条件的情况

         <img src="C:\Users\陈勇志的电脑\AppData\Roaming\Typora\typora-user-images\image-20221116205515086.png" alt="image-20221116205515086" style="zoom: 67%;" />

         <img src="C:\Users\陈勇志的电脑\AppData\Roaming\Typora\typora-user-images\image-20221116205848548.png" alt="image-20221116205848548" style="zoom:67%;" />

      2. 会触发过密密植条件的情况

         <img src="C:\Users\陈勇志的电脑\AppData\Roaming\Typora\typora-user-images\image-20221116210122183.png" alt="image-20221116210122183" style="zoom:67%;" />

         <img src="C:\Users\陈勇志的电脑\AppData\Roaming\Typora\typora-user-images\image-20221116210354545.png" alt="image-20221116210354545" style="zoom:67%;" />

      因此我们可以将Sg的计算总结为以下公式：
      $$
      Sg = (1+S_1+\Sigma S_i)\times(\frac{1}{2})^{k}
      $$
      $S_1$：附着干耕地 $S_1=1$；附着湿耕地 $S_1=3$；

      $S_i$：毗邻干耕地 $S_i=\frac{1}{4}$；毗邻湿耕地 $S_i=\frac{3}{4}$；$\Sigma$表示累加；

      $k$：毗邻小于等于一个方向有南瓜梗$k=1$；否则$k=0$

3. 计算被随机刻选中的南瓜梗生成南瓜（记为事件B）的概率
   $$
   P\left \{B\right \} = \frac{1}{int(\frac{25}{Sg}) + 1}
   $$
   注：int表示向下取整

4. 如果判断得到的是生成南瓜，则会在梗附近的四个方向随机选中一个方向生成南瓜，如果所选方向的方块不是空气或其下方不是泥土类方块，则会生成失败，且不会再去尝试在其他方向生成南瓜。

   设有效生成方向有n个，最多有4个方向可以选择，因此n<4，记最终生成南瓜的概率为Pf，则
   $$
   Pf = \frac{n}{4}P\{B\} = \frac{n}{4 (int(\frac{25}{Sg}) + 1)}
   $$

5. 接着我们可以求得在一小时，单个南瓜梗生成南瓜的产量

$$
S = E(T)|_{t=72000} \times Pf = 52.7\times\frac{n}{4 (int(\frac{25}{Sg}) + 1)}=13.18\frac{n}{int(\frac{25}{Sg}) + 1}
$$

#### 结论

通过上面的推导，我们可以总结单梗南瓜时产公式：
$$
S=13.18\frac{n}{int(\frac{25}{Sg}) + 1}
$$

$$
Sg = (1+S_1+\Sigma Si)\times(\frac{1}{2})^{k}
$$

$n$：有效生成方向的数量，且$n<4$；

$S_1$：附着干耕地 $S_1=1$；附着湿耕地 $S_1=3$；

$S_i$：毗邻干耕地 $S_i=\frac{1}{4}$；毗邻湿耕地 $S_i=\frac{3}{4}$；$\Sigma$表示累加；

$k$：毗邻小于等于一个方向有南瓜梗$k=0$；否则$k=1$。

根据公式我们可以看到南瓜的产量只与有效生成方向 $n$ 和生长速度 $Sg$ 来决定，而生长速度 $Sg$ 又由耕地数目及其湿润程度和相邻的同种梗确定。由此可以总结对南瓜生成的影响较大的因素有==光照强度==、==耕地数目==及其==湿润程度==、==相邻是否相同的梗==和==瓜生成方向==。

### 高效种植方案 

根据单梗南瓜时产公式，不难发现，要提高南瓜的产量，重点在增加湿润耕地数目、避免相邻相同的梗以及保证尽可能多的生成方向。

<img src="C:\Users\陈勇志的电脑\AppData\Roaming\Typora\typora-user-images\image-20221116124632259.png" alt="image-20221116124632259" style="zoom:50%;" />

如上图所示，当附着方块为湿润耕地、拥有8个湿润的毗邻耕地、毗邻方块没有相同梗、拥有最大四个有效生成方向，即 $n_{max}=4$ 和 $Sg_{max}=(1+3+8\times \frac{3}{4})\times (\frac{1}{2})^0=10$ 时，有着最大的理论单梗南瓜时产：
$$
S=13.18\frac{4}{int(\frac{25}{10}) + 1}=17.57
$$
当然，这种方案的单梗时产不可能维持下去，生成的南瓜会立刻使得生成方向的耕地变为泥土，使产量降低，最终形成下图所示的方案。

<img src="C:\Users\陈勇志的电脑\AppData\Roaming\Typora\typora-user-images\image-20221116131135576.png" alt="image-20221116131135576" style="zoom:50%;" />

该方案的 $n=4$ 、$Sg=(1+3+4\times \frac{3}{4})\times (\frac{1}{2})^0=7$ ，可计算得：
$$
S=13.18\frac{4}{int(\frac{25}{7}) + 1}=13.18
$$
13.18就是理论最大稳定单梗南瓜时产。



接着就是堆叠方案：

第一种堆叠方案如下图所示 ：

<img src="C:\Users\陈勇志的电脑\AppData\Roaming\Typora\typora-user-images\image-20221116131902160.png" alt="image-20221116131902160" style="zoom:100%;" />

该方案的单梗南瓜时产
$$
S = 13.18个/h
$$
为了方便比较，我们可以计算一下单位面积单梗南瓜时产。其最小组成单元如下图所示，水平方向占地4格（$4m^2$）

<img src="G:\Game\minecraft\_Other\工程\瓜机\图片\堆叠方案\2022-11-16_133642.png" alt="2022-11-16_133642" style="zoom:80%;" />

故单位面积单梗南瓜时产为：
$$
SdA = \frac{S}{4} = 3.30个/(m^2h)
$$
第二种堆叠方案如下图所示：

<img src="G:\Game\minecraft\_Other\工程\瓜机\图片\堆叠方案\2022-11-16_132239.png" alt="2022-11-16_132239" style="zoom:80%;" />

该方案的单梗南瓜时产：
$$
S = 6.59个/h
$$
最小组成单元如下图所示：

<img src="G:\Game\minecraft\_Other\工程\瓜机\图片\堆叠方案\2022-11-16_133528.png" alt="2022-11-16_133528" style="zoom:60%;" />

单位面积单梗南瓜时产为：
$$
SdA = \frac{S}{4} = 3.30个/(m^2h)
$$
通过对上面两种种植方式的比较，不难发现它们的单位面积单梗南瓜时产是相同的，但是第二种方案使用的南瓜梗的数量是第一种的两倍，单梗时产是第一种的一半，这主要因为第二种方案触发了过密种植的条件。此外第一种方案的剩余处的耕地可以用于种植西瓜，且不影响南瓜的产量。

最后我们看一下由李芒果设计，适用于服务器的南瓜种植方案[^3]，如下图所示

<img src="G:\Game\minecraft\_Other\工程\瓜机\图片\堆叠方案\2022-11-16_135509.png" alt="2022-11-16_135509" style="zoom:80%;" />

1号耕地上方种植其他作物或者使用地毯、线等覆盖，防止南瓜向这个方向生长，2、3、4号上方放水，便与收集且维持耕地湿润，5、6号方块放置可推动方块，7、8号泥土上方为南瓜的有效生成位置。 

容易得到该方案的 $n=2$ 、$Sg=(1+3+4\times \frac{3}{4})\times (\frac{1}{2})^0=7$ ，可计算得：
$$
S=13.18\frac{2}{int(\frac{25}{7}) + 1}=6.59
$$

### 高效种植方案的应用设计

下面是我根据稳定的高效种植方案设计的单梗瓜机。

<img src="C:\Users\陈勇志的电脑\AppData\Roaming\Typora\typora-user-images\image-20221115222612520.png" alt="image-20221115222612520" style="zoom:50%;" />



取其中一个方向来看，侦测器侦测南瓜梗的状态，当南瓜梗生成南瓜后，侦测器会检测到南瓜梗的变化，对白色方块充能，使活塞处于bud状态，白色方块也会激活充能铁轨，充能铁轨发出NC更新使活塞推出，将南瓜推掉。当然如果条件允许，可以将充能铁轨换为音符盒，方便施工。

<img src="C:\Users\陈勇志的电脑\AppData\Roaming\Typora\typora-user-images\image-20221115222933797.png" alt="image-20221115222933797" style="zoom:50%;" />



接着将它们进行堆叠至一个区块内，效果如下图所示，堆叠方案采用的是上面第一种方案。空余的耕地可以用于种植西瓜。

![2022-11-16_145855](G:\Game\minecraft\_Other\工程\瓜机\图片\堆叠方案\2022-11-16_145855.png)

其种植示意图如下所示：

<img src="G:\Game\minecraft\_Other\工程\瓜机\图片\堆叠方案\2022-11-16_180157.png" alt="2022-11-16_180157" style="zoom:100%;" />

浅黄色方块表示种植南瓜，浅绿色方块表示种植西瓜，蓝色方块为水源，白色位置为瓜的生长位置，橙色为光源，这里使用的是红石灯，通过控制红石灯的亮灭来实现瓜机的开关。

由于无法使用水道来进行收集，只能采用漏斗矿车收集。

综上设计出单区块可纵向堆叠瓜机单片：

![2022-11-16_153457](G:\Game\minecraft\_Other\工程\瓜机\图片\堆叠方案\2022-11-16_153457.png)

![2022-11-16_153547](G:\Game\minecraft\_Other\工程\瓜机\图片\堆叠方案\2022-11-16_153547.png)



单片理论产量计算：

<img src="G:\Game\minecraft\_Other\工程\瓜机\图片\堆叠方案\2022-11-16_180157.png" alt="2022-11-16_180157" style="zoom:100%;" />

由于受到空间、光源、水源以及收集装置的限制，靠近边缘、光源和水源的瓜梗生产效率或多或少的会降低，考虑进这些因素，且忽略瓜生成后变成掉落物的时间。

理论南瓜产量计算表格：

| 排列方式 | 第一行 | 第二行 | 第三行 | 第四行 | 第五行 | 第六行 | 第七行 | 总计 | 单产  |
| :------: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :--: | :---: |
|   4-4    |   3    |   3    |   6    |   4    |   6    |   3    |        |  25  | 13.18 |
|   4-3    |        |   1    |        |   2    |        |   2    |        |  5   | 9.89  |
|   3-4    |   2    |   2    |        |        |        |        |        |  4   | 10.54 |
|   2-3    |   1    |   1    |   1    |   1    |   1    |   1    |   5    |  11  | 7.91  |
|   2-2    |   1    |        |        |        |        |        |   1    |  2   | 5.27  |

注：

1. 排列方式中第一个数表示毗邻湿润耕地数目，第二个数表示有效方向数目
2. 第n行表示从上往下第n行南瓜对应排列方式的南瓜梗数目
3. 单产表示对应排列方式单梗时产。

理论南瓜产量：
$$
Spt = 25*13.18+5*9.89+4*10.54+11*7.91+2*5.27 = 518.66个/h
$$
通过五个小时的测试，实测南瓜产量如下图所示：

![image-20221116164929294](C:\Users\陈勇志的电脑\AppData\Roaming\Typora\typora-user-images\image-20221116164929294.png)

实测产量为：
$$
Spr = 517.6个/h
$$
理论和实测的数据基本吻合。

可以看到，单区块单个瓜机单片有这实测约517.6个/h的南瓜产量和2713.9个/h的西瓜片产量，算的平均每个梗的产量为南瓜$S = 517.6/47=11.01个/h$、西瓜片$S = 2713.9/46 = 60.00个/h$。

### 瓜机方案的优缺点

该瓜机方案有着以下优点：

1. 效率高。根据高效方案设计出的瓜机有着高达平均11.01个/h的单梗南瓜产量，并且能够额外附加60.00个/h的单梗西瓜产量。
2. 体积小。相较于李芒果设计的瓜机，每个梗所占用的体积约缩小20%，生产率提高约66.9%（不考虑南瓜和西瓜混种）。
3. 单区块。每层占用6格高度，可纵向堆叠，当然也可以改进改进，采用水平方向堆叠。
4. 开关机可控。通过红石灯的亮灭来控制作物生长的光照环境，进而实现瓜机的开关。

当然，缺点也很明显：

1. 不抗卸载。由于收集采用的是漏斗矿车收集，故此瓜机是不抗卸载的，这就带来一个很严重的问题，当单片堆叠数量很多时，如果矿车因为区块卸载而停止运行，维护矿车会非常麻烦。
2. 不适合服务器。首先服务器一般需要高的产量，就需要堆叠许多瓜机单片，不仅漏斗矿车维护很不方便，而且使用较多的漏斗矿车是服务器卡顿的来源之一。此外，我设计瓜机时用到了t触发器，t触发器在服务器中运行不稳定，容易错位，在单人存档中则不会存在此问题。
3. 不建议用于1.14之前的版本。1.14版本添加了玩家趴下的动作，在瓜机的建造和漏斗矿车的维护过程中需要玩家趴下建造，因此不建议用于1.14之前的版本。

综上，该瓜机方案比较推荐单人存档使用，可以根据个人需求确定瓜机单片堆叠层数。不建议堆叠过高层数。

此外，当玩家距离南瓜梗距离较远时，南瓜梗结出南瓜后有概率没有恢复原状，如下图所示：

<img src="C:\Users\陈勇志的电脑\AppData\Roaming\Typora\typora-user-images\image-20221116203907539.png" alt="image-20221116203907539" style="zoom: 33%;" />

结果的南瓜梗是不会再结出果实的，这会使得效率降低。这也是不建议堆叠过多单片的原因之一。

### 参考文献

[^1]:  [刻 - Minecraft Wiki，最详细的我的世界百科 (fandom.com)](https://minecraft.fandom.com/zh/wiki/刻)
[^2]: [Minecraft1.17源码获取来源 GitHub - Hexeption/MCP-Reborn: MCP-Reborn is an MCP (Mod Coder Pack) for Minecraft for making modded clients and researching its code. (1.13-1.19.2)](https://github.com/Hexeption/MCP-Reborn)
[^3]: [【MC|熟肉】廉价可堆叠简单不卡顿高效西瓜南瓜机（几乎无损失）（填坑）【ilmango】](https://www.bilibili.com/video/BV1Zs411v7yb); [cheap, simple, stackable, lag friendly, efficient melon/pumpkin farm (almost lossless) - YouTube](https://www.youtube.com/watch?v=45zVsJOFBzY)

[^另1]: [南瓜 - Minecraft Wiki，最详细的我的世界百科 (fandom.com)](https://minecraft.fandom.com/zh/wiki/南瓜)
[^另2]: [教程/西瓜和南瓜种植 - Minecraft Wiki，最详细的我的世界百科 (fandom.com)](https://minecraft.fandom.com/zh/wiki/教程/西瓜和南瓜种植)

