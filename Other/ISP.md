## ISP


|              |                                 |                       |                       |
| :----------- | :------------------------------ | :-------------------- | --------------------- |
| 串行器       | 解串器                          |                       |                       |
| GMSL1        | MAX96701/MAX96715               | MAX9286               |                       |
|              | 12bit DVP IN 1 SIO OUT@1.76Gbps | 4IN 1OUT@MIPI 1.5Gbps |                       |
| GMSL1/2 兼容 | MAX9295A                        | MAX9296               | MAX96722              |
|              | 4-MIPI IN 1 SIO OUT@6Gbps       | 2IN 2OUT@MIPI 2.5Gbps | 4IN 1OUT@MIPI 2.5Gbps |
| GMSL2        | MAX96717/MAX96717F              | MAX96712              |                       |
|              | 4-MIPI IN 1 SIO OUT@6Gbps/3Gbps | 4IN 1OUT@MIPI 2.5Gbps |                       |

 ISP(Image Signal Processor)，即图像处理，主要作用是对前端图像传感器输出的信号做后期处理，主要功能有线性纠正、噪声去除、坏点去除、内插、白平衡、自动曝光控制等，依赖于ISP才能在不同的光学条件下都能较好的还原现场细节，ISP技术在很大程度上决定了摄像机的成像质量。它可以分为独立与集成两种形式。
![img](https://img-blog.csdn.net/20180520013530288)

ISP 的Firmware 包含三部分，一部分是ISP 控制单元和基础算法库，一部分是AE/AWB/AF 算法库，一部分是sensor 库。Firmware 设计的基本思想是单独提供3A 算法库，由ISP 控制单元调度基础算法库和3A 算法库，同时sensor 库分别向ISP 基础算法库和3A 算法库注册函数回调，以实现差异化的sensor 适配。ISP firmware 架构如下图所示。

![img](https://img-blog.csdn.net/2018052001372125)

不同的sensor 都以回调函数的形式，向ISP 算法库注册控制函数。ISP 控制单元调度基础算法库和3A 算法库时，将通过这些回调函数获取初始化参数，并控制sensor，如调节曝光时间、模拟增益、数字增益，控制lens 步进聚焦或旋转光圈等。

#### 1.TestPattern------测试图像

Test Pattern主要用来做测试用。不需要先在片上ROM存储图片数据，直接使用生成的测试图像，用生成的测试图像进行后续模块的测试验证。以下是常用的两种测试图像。

![img](https://img-blog.csdn.net/20170503224924192?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHowNDk5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### 2.BLC(BlackLevel Correction)------黑电平校正

​    Black Level 是用来定义图像数据为 0 时对应的信号电平。由于暗电流的影响， 传感器出来的实际原始数据并不是我们需要的黑平衡（ 数据不为0） 。 所以，为减少暗电流对图像信号的影响，可以采用的有效的方法是从已获得的图像信号中减去参考暗电流信号，或者更确切是：模拟信号很微弱的时候，有可能不能被A/D转换出来，导致光线很暗的时候，图像暗区细节丢失。因此，sensor一般会在A/D转换之前，给模拟信号一个偏移量，以确保输出的图像保留足够多的细节。而黑电平校正主要是通过标定的方式确定这个偏移量。使得后续ISP模块的处理在保持线性一致性的基础上进行。

```
 一般情况下， 在传感器中，实际像素要比有效像素多， 像素区头几行作为不感光区（ 实际上， 这部分区域也做了 RGB 的 color filter） ， 用于自动黑电平校正， 其平均值作为校正值， 然后在下面区域的像素都减去此矫正值， 那么就可以将黑电平矫正过来了。如下图所示，左边是做黑电平校正之前的图像，右边是做了黑电平校正之后的图像。

 黑电平校正是在一倍系统增益的情况下标定计算而来，有些sensor在高倍增益和低倍增益时，OB相差会比较大。这个时候就需要获取不同增益环境下的遮黑RAW数据，分析R/Gr/Gb/B四个通道下的mean值。分析出来的均值即为各个通道的OB值。如果需要微调，即可在标定的OB上进行。例如：低照度下偏蓝，即可根据所在的ISO范围将B通道的幅度增加，减轻偏蓝现象。
```

![img](https://img-blog.csdn.net/20170503225104460?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHowNDk5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### 3.LSC(Lens Shade Correction)------镜头阴影校正

​     由于相机在成像距离较远时，随着视场角慢慢增大，能够通过照相机镜头的斜光束将慢慢减少，从而使得获得的图像中间比较亮，边缘比较暗，这个现象就是光学系统中的渐晕。由于渐晕现象带来的图像亮度不均会影响后续处理的准确性。因此从图像传感器输出的数字信号必须先经过镜头矫正功能块来消除渐晕给图像带来的影响。同时由于对于不同波长的光线透镜的折射率并不相同，因此在图像边缘的地方，其R、G、B的值也会出现偏差，导致CA(chroma aberration)的出现，因此在矫正渐晕的同时也要考虑各个颜色通道的差异性。

    常用的镜头矫正的具体实现方法是，首先确定图像中间亮度比较均匀的区域，该区域的像素不需要做矫正；以这个区域为中心，计算出各点由于衰减带来的图像变暗的速度，这样就可以计算出相应R、G、B通道的补偿因子(即增益)。下图左边图像是未做镜头阴影校正的，右边图像是做了镜头阴影校正的。
    
    出于节约成本的考虑以及尺寸方面的原因，手机相机镜头向小型化和低成本方向发展。由于摄像头尺寸小，制造材料品质低，拍摄的图像在靠近边缘处会出现亮度衰减的现象。因此要对 Bayer raw 图像进行镜头衰减校正，以降低计算负荷。使用 LUT 分段线性近似法代替模拟曲线和多项式运算。每种颜色都有自己的 LUT，因此亮度衰减和色偏问题可同时得到解决。

   针对不同增益下的LSC校正强度也会有所不一样。低照度下相对会比正常光照情况下校正强度要小一些。因此，ISP会预留接口以便对不同增益下的LSC强度进行调整。抑或者预留接口控制图像不同区域的LSC校正强度。例如：从中心区域开始往图像四周校正强度逐级减弱。

![img](https://img-blog.csdnimg.cn/20201215185100976.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2x6MDQ5OQ==,size_16,color_FFFFFF,t_70)

                                                    LSC校准原理

![img](https://img-blog.csdn.net/20170503225323322?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHowNDk5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

                                              LSC校准前后结果对比

4.DPC(Bad Point Correction)------坏点校正
     所谓坏点，是指像素阵列中与周围像素点的变化表现出明显不同的像素，因为图像传感器是成千上万的元件工作在一起，因此出现坏点的概率很大。一般来讲，坏点分为三类：第一类是死点，即一直表现为最暗值的点；第二类是亮点，即一直表现为最亮值的点：第三类是漂移点，就是变化规律与周围像素明显不同的像素点。由于图像传感器中CFA的应用，每个像素只能得到一种颜色信息，缺失的两种颜色信息需要从周围像素中得到。如果图像中存在坏点的话，那么坏点会随着颜色插补的过程往外扩散，直到影响整幅图像。因此必须在颜色插补之前进行坏点的消除。

    盐椒噪声是一种在图像中产生黑点或白点的脉冲噪声，这类噪声往往和图像信号内容不相关，与邻域周边像素灰度值差别明显。中值滤波能够较好的滤除盐椒噪声（冲激噪声）。对于Sensor坏点来说，在一定程度上也可以看做是盐椒噪声，因此，坏点校正也可以使用中值滤波进行滤除。

   算法基本原理：

![img](https://img-blog.csdnimg.cn/20210706215653592.png)

      以图1．10中P4点为例，坏点消除基本过程为：首先计算该像素点与周围像素点像素值的差：

![img](https://img-blog.csdnimg.cn/20210706215717513.png)

      设定一个阈值，作为判断的标准。判断各个方向上的差值跟阈值的关系，如果都大于阈值的话，就表明该点像素值与周围像素点的差别较大，就可以确定该像素点为坏点，否则该像素就为正常的像素点，可以进行下一个像素点的处理。
    
      若判断出某点为坏点，接下来进行对其的校正，过程如下：计算该点各个方向上的导数：

![img](https://img-blog.csdnimg.cn/20210706215807438.png)

     确定出该值最小的方向，表明该像素点需要在该方向上进行补偿，则按照下面的公式对该像素进行补偿，即若假设有

((DV<=DH) && (DV<DDL)&&(DV<=DDR))

    则该像素点的值可以用下面的值来取代 

Avg = (p1+2*p4+p7)/4

参考如下：isp 图像算法(二)之dead pixel correction坏点矫正

![img](https://img-blog.csdnimg.cn/92d5ede8a7ac413298f5e7d6f9dad27b.png)

#!/usr/bin/python
import numpy as np

class DPC:
    'Dead Pixel Correction'

    def __init__(self, img, thres, mode, clip):
        self.img = img
        self.thres = thres
        self.mode = mode
        self.clip = clip
     
    def padding(self):
        #在四周放两个0 从(1080,1920) --->(1084,1924)
        img_pad = np.pad(self.img, (2, 2), 'reflect')
        return img_pad
     
    def clipping(self):
        
        #np.clip是一个截取函数，用于截取数组中小于或者大于某值的部分，并使得被截取部分等于固定值
        #限定在()0,1023
        np.clip(self.img, 0, self.clip, out=self.img)
        return self.img
     
    def execute(self):
        img_pad = self.padding()
        raw_h = self.img.shape[0]
        raw_w = self.img.shape[1]
        dpc_img = np.empty((raw_h, raw_w), np.uint16)
        for y in range(img_pad.shape[0] - 4):
            for x in range(img_pad.shape[1] - 4):
                p0 = img_pad[y + 2, x + 2]
                p1 = img_pad[y, x]
                p2 = img_pad[y, x + 2]
                p3 = img_pad[y, x + 4]
                p4 = img_pad[y + 2, x]
                p5 = img_pad[y + 2, x + 4]
                p6 = img_pad[y + 4, x]
                p7 = img_pad[y + 4, x + 2]
                p8 = img_pad[y + 4, x + 4]
                if (abs(p1 - p0) > self.thres) and (abs(p2 - p0) > self.thres) and (abs(p3 - p0) > self.thres) \
                        and (abs(p4 - p0) > self.thres) and (abs(p5 - p0) > self.thres) and (abs(p6 - p0) > self.thres) \
                        and (abs(p7 - p0) > self.thres) and (abs(p8 - p0) > self.thres):
                    if self.mode == 'mean':
                        p0 = (p2 + p4 + p5 + p7) / 4
                    elif self.mode == 'gradient':
                        dv = abs(2 * p0 - p2 - p7)
                        dh = abs(2 * p0 - p4 - p5)
                        ddl = abs(2 * p0 - p1 - p8)
                        ddr = abs(2 * p0 - p3 - p6)
                        if (min(dv, dh, ddl, ddr) == dv):
                            p0 = (p2 + p7 + 1) / 2
                        elif (min(dv, dh, ddl, ddr) == dh):
                            p0 = (p4 + p5 + 1) / 2
                        elif (min(dv, dh, ddl, ddr) == ddl):
                            p0 = (p1 + p8 + 1) / 2
                        else:
                            p0 = (p3 + p6 + 1) / 2
                dpc_img[y, x] = p0
        self.img = dpc_img
        return self.clipping()

 


#### 5.GB（Green Balance）------绿平衡

​    由于感光器件制造工艺和电路问题，Gr,Gb数值存在差异,将出现格子迷宫现象可使用均值算法处理Gr,Gb通道存在的差异,同时保留高频信息。

   另外一个说法是：

    Sensor芯片的Gr，Gb通道获取的能量或者是输出的数据不一致，造成这种情况的原因之一是Gr，GB通道的半导体制造工艺方面存在差异，另一方面是Microlens的存在，特别是sensor边缘区域，GB，Gr因为有角度差异，导致接收到的光能不一致。如果两者差异比较大，就会出现类似迷宫格子情况。主要是考虑G周围的G的方法进行平均化

![img](https://img-blog.csdnimg.cn/2020121518494821.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2x6MDQ5OQ==,size_16,color_FFFFFF,t_70)

                                                        Optical cross-talk示意图


![img](https://img-blog.csdn.net/20170503225609071?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHowNDk5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

  算法基本原理：

![img](https://img-blog.csdnimg.cn/20210627221400758.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2x6MDQ5OQ==,size_16,color_FFFFFF,t_70)

*A 5x5 local window from Bayer GRBG pattern, (a) The central pixel G7 is Gr; (b) the central pixel G7 is Gb*

* 基于插值的方法
  该算法中，选取Gr或Gb为参考颜色通道，修改另一个G通道分量，使得Gr/Gb两通道的数值基本一致。

假设Gb作为参考通道a图中的位于位置7的Gr像素值应该按照如下公式修改：

![img](https://img-blog.csdnimg.cn/20210627221719436.JPG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2x6MDQ5OQ==,size_16,color_FFFFFF,t_70)

 如果选择Gr为参考通道，则只需要按照同样的方法修改Gb的像素值即可

* 基于平均值的方法
  该算法中Gr，Gb两个通道中的像素值都需要修改。

图a中，对于位于7位置的Gr,需要按照如下公式修：

![img](https://img-blog.csdnimg.cn/20210627221859823.JPG)

 图b中，对于位于7位置的Gb,需要按照如下公式修改：

![img](https://img-blog.csdnimg.cn/20210627221935627.JPG)

#### 6.Denoise-----去除噪声

​    使用 cmos sensor 获取图像，光照程度和传感器问题是生成图像中大量噪声的主要因素。同时， 当信号经过 ADC 时， 又会引入其他一些噪声。 这些噪声会使图像整体变得模糊， 而且丢失很多细节， 所以需要对图像进行去噪处理空间去噪传统的方法有均值滤波、 高斯滤波等。

    但是， 一般的高斯滤波在进行采样时主要考虑了像素间的空间距离关系， 并没有考虑像素值之间的相似程度， 因此这样得到的模糊结果通常是整张图片一团模糊。 所以， 一般采用非线性去噪算法， 例如双边滤波器， 在采样时不仅考虑像素在空间距离上的关系， 同时加入了像素间的相似程度考虑， 因而可以保持原始图像的大体分块， 进而保持边缘。

   关于2D denoise可以参考：一种基于bayer型模式的双边自适应滤波器

#### 7.Demosaic------颜色插值

​    光线中主要包含三种颜色信息，即R、G、B。但是由于像素只能感应光的亮度，不能感应光的颜色，同时为了减小硬件和资源的消耗，必须要使用一个滤光层，使得每个像素点只能感应到一种颜色的光。目前主要应用的滤光层是bayer GRBG格式。如下图所示：

![img](https://img-blog.csdn.net/20170503225814063?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHowNDk5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

    这样，经过滤色板的作用之后，每个像素点只能感应到一种颜色。必须要找到一种方法来复原该像素点其它两个通道的信息，寻找该点另外两个通道的值的过程就是颜色插补的过程。由于图像是连续变化的，因此一个像素点的R、G、B的值应该是与周围的像素点相联系的，因此可以利用其周围像素点的值来获得该点其它两个通道的值。目前最常用的插补算法是利用该像素点周围像素的平均值来计算该点的插补值。如下图所示，左侧是RAW域原始图像，右侧是经过插值之后的图像。

![img](https://img-blog.csdn.net/20170503230012714?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHowNDk5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### 8.AWB（Automatic White Balance）------自动白平衡

​    人类视觉系统具有颜色恒常性的特点，因此人类对事物的观察可以不受到光源颜色的影响。但是图像传感器本身并不具有这种颜色恒常性的特点，因此，其在不同光线下拍摄到的图像，会受到光源颜色的影响而发生变化。例如在晴朗的天空下拍摄到的图像可能偏蓝，而在烛光下拍摄到的物体颜色会偏红。因此，为了消除光源颜色对于图像传感器成像的影响，自动白平衡功能就是模拟了人类视觉系统的颜色恒常性特点来消除光源颜色对图像的影响的。

关于自动白平衡相关知识可以参考：理解ISP自动白平衡标定 、ISP——AWB(Auto White Balance)

![img](https://img-blog.csdn.net/20170503230205319?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHowNDk5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### 9.CCM（Color Correction Matrix）------颜色校正

​    颜色校正主要为了校正在滤光板处各颜色块之间的颜色渗透带来的颜色误差。一般颜色校正的过程是首先利用该图像传感器拍摄到的图像与标准图像相比较，以此来计算得到一个校正矩阵。该矩阵就是该图像传感器的颜色校正矩阵。在该图像传感器应用的过程中，及可以利用该矩阵对该图像传感器所拍摄的所有图像来进行校正，以获得最接近于物体真实颜色的图像。

    一般情况下，对颜色进行校正的过程，都会伴随有对颜色饱和度的调整。颜色的饱和度是指色彩的纯度，某色彩的纯度越高，则其表现的就越鲜明；纯度越低，表现的则比较黯淡。RGB三原色的饱和度越高，则可显示的色彩范围就越广泛。 

   一般在不同增益环境下CCM的饱和度会有所不同。例如，低照度小我们可以适当调低CCM饱和度以减轻低照度下色噪。因此，一般ISP会留出接口以便对不同增益下CCM饱和度调整，对一倍增益校正出的CCM参数进行插值计算，计算得到不同增益下较为合适的CCM参数。

        手动调整CCM参数原理可以参考：Color correction matrix（色彩矩阵）的学习思考

![img](https://img-blog.csdn.net/20170503230439635?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHowNDk5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### 10.RGB Gamma------Gamma校正

​    伽马校正的最初起源是CRT屏幕的非线性，研究CRT电子枪的物理表明，电子枪的输入电压和输出光之间满足5．2幂函数关系，即荧光屏上显示的亮度正比于输入电压的5／2次方，这个指数被称为伽马。这种关系源于阴极、光栅和电子束之间的静电相互作用。由于对于输入信号的发光灰度，不是线性函数，而是指数函数，因此必需校正。Gamma校正的硬件实现方式可以参考： 一种基于分段线性插值的Gamma校正硬件实现和Adaptive piece-wise approximation method for gamma correction

    但是实际情况是，即便CRT显示是线性的，伽马校正依然是必须的，是因为人类视觉系统对于亮度的响应大致是成对数关系的，而不是线性的。人类视觉对低亮度变化的感觉比高亮度变化的感觉来的敏锐，当光强度小于1lux时，常人的视觉敏锐度会提高100倍t2118]。伽马校正就是为了校正这种亮度的非线性关系引入的一种传输函数。校正过程就是对图像的伽玛曲线进行编辑，检出图像信号中的深色部分和浅色部分，并使两者比例增大，从而提高图像对比度效果，以对图像进行非线性色调编辑。由于视觉环境和显示设备特性的差异，伽马一般取2．2～2．5之间的值。当用于校正的伽马值大于1时，图像较亮的部分被压缩，较暗的部分被扩展；而伽马值小于1时，情况则刚好相反。
    
    现在常用的伽马校正是利用查表法来实现的，即首先根据一个伽马值，将不同亮度范围的理想输出值在查找表中设定好，在处理图像的时候，只需要根据输入的亮度，既可以得到其理想的输出值。在进行伽马校正的同时，可以一定范围的抑制图像较暗部分的噪声值，并提高图像的对比度。还可以实现图像现显示精度的调整，比如从l0bit精度至8bit精度的调整。上图分别是未做Gamma校正的，下图是做了Gamma校正的。

![img](https://img-blog.csdn.net/20170503231134097?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHowNDk5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



#### 11.RGBToYUV

​    YUV 是一种基本色彩空间， 人眼对亮度改变的敏感性远比对色彩变化大很多， 因此， 对于人眼而言， 亮度分量 Y 要比色度分量 U、 V 重要得多。 另外，YUV色彩空间分为YUV444,YUV422，YUV420等格式，这些格式有些比原始RGB图像格式所需内存要小很多，这样亮度分量和色度分量分别存储之后，给视频编码压缩图像带来一定好处。

      RGB转YUV可以参考：色彩转换系列之RGB格式与YUV格式互转原理及实现

#### 12.WDR（Wide Dynamic Range）------宽动态

​    动态范围(Dynamic Range)是指摄像机支持的最大输出信号和最小输出信号的比值，或者说图像最亮部分与最暗部分的灰度比值。普通摄像机的动态范围一般在1:1000(60db)左右，而宽动态(Wide Dynamic Range,WDR)摄像机的动态范围能达到1:1800-1:5600(65-75db)。

　　宽动态技术主要用来解决摄像机在宽动态场景中采集的图像出现亮区域过曝而暗区域曝光不够的现象。简而言之，宽动态技术可以使场景中特别亮的区域和特别暗的区域在最终成像中同时看清楚。

![img](https://img-blog.csdn.net/20170503231852645?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHowNDk5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### 13.3DNR

​         3dnr 是结合空域滤波和时域滤波的一种降噪算法。大概思路是检测视频的运动水平，更具运动水平的大小对图像像素进行空域滤波和时域滤波的加权，之后输出滤波之后的图像。

        基本原理可以参考：运动自适应降噪_Motion Adaptive Noise Reduction

![img](https://img-blog.csdnimg.cn/20200409212847774.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2x6MDQ5OQ==,size_16,color_FFFFFF,t_70)

参考：噪声及降噪算法总结、适用于3DNR图像降噪的raw图像压缩方法及装置

    空域降噪 是针对单帧画面进行处理 ；时域降噪 是结合前后帧进行计算处理的出来的 。
    
    一般来说，降噪我们都会在最前面的节点，对原始素材进行降噪。
    
    如果我们只用空域降噪的话，假如我们连续播放的话，由于它的算法都是基于当前帧，不考虑前后帧，所以这样很有可能会发生一些噪点的抖动。
    
    时域降噪会把整个时间段的噪点统一进行处理，消除帧与帧之间的噪点抖动。目前达芬奇14支持最高的是计算前后五帧。当然选择帧数越多，那对计算机的性能要求越高。
    
    亮度阈值简单的说就是控制亮度噪点。如果数值设置太高，会消除一些图像的细节。色度阈值就是控制色彩噪点。同样的数值设置的太高的话，颜色细节也会消失不见。混合数值就是控制降噪的强度。可以把它理解成透明度，也透明影响越小。越不透明，影响越大。
    
    动作阈值就是你判断你画面中大概有百分之多少的像素，来判断出我们大概使用多少范围的运动。超出这个阈值的像素是运动的，低于这个阈值的像素是静态的。数值如果过高就会产生运动的残留或者是残影。一般都是先利用空域降噪对单帧进行一个基础的降噪，然后在利用时域降噪把与帧之间的噪点抖动消除，这样的效果可能会比较好。
    
    此外，3dnr是一种时域与空域相结合的图像去噪方法，在isp芯片对视频的处理过程中，3dnr去噪的时域模块在对当前帧去噪时往往需要与上一帧（即对比帧）进行比较以获取最优的去噪效果。因此，isp芯片在设计3dnr去噪模块的同时会留出一块内存以存储对比帧数据。而一张1920*1080的12bit宽动态图像需要3mb左右的存储空间，这对芯片的成本控制是极大的负担，所以在isp中还需增加图像压缩模块尽可能减少存储对比帧所需的数据量。
    由于在3dnr去噪过程中对对比帧的精确度要求较高，因此该压缩模块所使用的压缩算法必须为无损压缩算法。传统图像无损压缩方法中，无损预测编码易于数字电路的设计，并且对rgb图像有不错的压缩率，但是对3dnr时域去噪后的raw对比帧数据的压缩效果微乎甚微。

![img](https://img-blog.csdnimg.cn/d54e2a5c371e454a86419e2b44de0253.png)

#### 14.Sharp------锐化

​    CMOS输入的图像将引入各种噪声，有随机噪声、量化噪声、固定模式噪声等。ISP降噪处理过程中，势必将在降噪的同时，把一些图像细节给消除了，导致图像不够清晰。为了消除降噪过程中对图像细节的损失，需要对图像进行锐化处理，还原图像的相关细节。如下图所示，左图是未锐化的原始图像，右图是经过锐化之后的图像。

    为了避免把Noise enhance出来，sharp在实现中还需要判断当前像素处于光滑区域还是物体边缘。当处于光滑区域的时候，则不要做sharp运算，或者做的幅度很小；只有在较明显的边缘上才做处理，这样避免不了边缘上的noise的影响，所以在锐利度s设定较大的时候，可以发现边缘上会有Noise闪动跳跃的情况。为了缓解这种Noise的跳跃，通常会对f(d, g, s)做最大值和最小值限制保护，并且沿着edge 方向做低通滤波来缓解Noise。

   sharp大致流程为先判断平坦区域还是边缘，对平坦区域可以不做或者少做sharp(甚至做smooth处理)，对边缘要判断幅度大小，边缘方向，选则相对应的高通滤波器处理，最后对enhance的幅度做一定程度的保护处理。

![img](https://img-blog.csdn.net/20170503232126751?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHowNDk5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

sharp算法基本原理（节选于《基于DSP的网络摄像机图像预处理技术》原文表述有些问题，已经按照作者想表达的意思进行的修改）：

边缘增强模块对图像的亮度分量(Y数据)进行操作来增强图像质量。首先按固定系数2D线性滤波器滤波计算边缘锐度sharpness(h，v)。sharp_shrink之后还可以结合查找表改变其锐化强度，然后再进行shoot的限制操作。

![img](https://img-blog.csdnimg.cn/20210628222652437.JPG)	

clip与shrink函数表达式如下：

![img](https://img-blog.csdnimg.cn/20210628222734170.JPG)

![img](https://img-blog.csdnimg.cn/2021062822274558.JPG)



 两者函数图像如下图所示：

![img](https://img-blog.csdnimg.cn/20210628222810204.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2x6MDQ5OQ==,size_16,color_FFFFFF,t_70)

      shrink函数把高频锐化结果靠近0的部分都强制设为0，即比较平滑的部分不做锐化处理；远离0的部分适当减小其锐化强度。当高频锐化结果x比th比较大或比较小，如果x比th较大，那么适当在x的基础上减小一点，减低白边的锐化强度；如果x比-th较小，那么在x的基础上适当增加一点，降低黑边的锐化强度 。

![img](https://img-blog.csdnimg.cn/2021062822342635.JPG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2x6MDQ5OQ==,size_16,color_FFFFFF,t_70)

锐化前表现

![img](https://img-blog.csdnimg.cn/20210628223448918.JPG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2x6MDQ5OQ==,size_16,color_FFFFFF,t_70)

锐化后表现

#### 15.AF（Automatic Focus）----自动对焦

​    自动对焦方面知识，可参考：

自动对焦模块理论基础及其硬件实现浅析（一）
自动对焦模块理论基础及其硬件实现浅析（二）
自动对焦模块理论基础及其硬件实现浅析（三）
自动对焦模块理论基础及其硬件实现浅析（四）

#### 16.AE（Automatic Exposure）----自动曝光

​    不同场景下，光照的强度有着很大的差别。人眼有着自适应的能力因此可以很快的调整，使自己可以感应到合适的亮度。而图像传感器却不具有这种自适应能力，因此必须使用自动曝光功能来确保拍摄的照片获得准确的曝光从而具有合适的亮度。

    AE 模块实现的功能是：根据自动测光系统获得当前图像的曝光量，再自动配置镜头光圈、sensor快门及增益来获得最佳的图像质量。自动曝光的算法主要分光圈优先、快门优先、增益优先。光圈优先时算法会优先调整光圈到合适的位置，再分配曝光时间和增益，只适合p-iris 镜头，这样能均衡噪声和景深。快门优先时算法会优先分配曝光时间，再分配sensor增益和ISP 增益，这样拍摄的图像噪声会比较小。增益优先则是优先分配sensor增益和ISP 增益，再分配曝光时间，适合拍摄运动物体的场景。基本算法原理可以参考：3A+ISP之AE篇
    
    自动曝光的实现一般包括三个步骤：光强测量、场景分析和曝光补偿。光强测量的过程是利用图像的曝光信息来获得当前光照信息的过程。按照统计方式的不同，分为全局统计，中央权重统计或者加权平均统计方式等。全局统计方式是指将图像全部像素都统计进来，中央权重统计是指只统计图像中间部分，这主要是因为通常情况下图像的主体部分都位于图像的中间部分；加权平均的统计方式是指将图像分为不同的部分，每一部分赋予不同的权重，比如中间部分赋予最大权重，相应的边缘部分则赋予较小的权重，这样统计得到的结果会更加准确。场景分析是指为了获得当前光照的特殊情况而进行的处理，比如有没有背光照射或者正面强光等场景下。对这些信息的分析，可以提升图像传感器的易用性，并且能大幅度提高图像的质量，这是自动曝光中最为关键的技术。目前常用的场景分析的技术主要有模糊逻辑和人工神经网络算法。这些算法比起固定分区测光算法具有更高的可靠性，主要是因为在模糊规则制定或者神经网络的训练过程中已经考虑了各种不同光照条件。在完成了光强测量和场景分析之后，就要控制相应的参数使得曝光调节生效。主要是通过设定曝光时间和曝光增益来实现的。通过光强测量时得到的当前图像的照度和增益值与目标亮度值的比较来获得应该设置的曝光时间和增益调整量。在实际情况下，相机通常还会采用镜头的光圈/快门系统来增加感光的范围。
    
    在进行曝光和增益调整的过程中，一般都是变步长来调整的，这样可以提高调整的速度和精度。一般来讲，增益和曝光的步长设定如下图所示：

![img](https://img-blog.csdn.net/20170503232222440?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHowNDk5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

    从上图中可以看出，在当前曝光量与目标量差别在range0以内的时候，说明当前曝光已经满足要求，不需要进行调整；差别在rangel的范围内时，则说明当前曝光与要求的光照有差别，但差别不大，只需要用较小的步长来进行调节即可；当差别在range2的时候，则表明差别较大，需要用较大步长来进行调节。在实现过程中还需要注意算法的收敛性。

   实际应用中，AE曝光时间和增益的调整大致如下图所示：

![img](https://img-blog.csdnimg.cn/20210209214004169.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2x6MDQ5OQ==,size_16,color_FFFFFF,t_70)

 弱曝、过曝、曝光合适实际效果图：

![img](https://img-blog.csdn.net/20170503232540633?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHowNDk5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![img](https://img-blog.csdn.net/20170503232440569?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHowNDk5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

      由于在对视频流做处理时，有些操作往往不是立即生效的。例如在自动曝光的处中，需要计算全局的亮度平均值。由于这个过程涉及到一帧中的所有像素点，所以在一帧图像输出完成之后才能得到亮度平均值。那么自动曝光所得到当前帧的计算结果，只能去调节下一帧的亮度，而无法影响当前帧。这个现象很普遍，只要程序存在全局的参数，就不可能在当前帧中得到计算结果。所以需要将第N帧数据计算出的参数或是结果，传递给第N+1帧，在第N+1帧中直接使用这个参数进行其他的计算，或者直接输出调整后的结果，我们将这种方法叫做帧迭代方法。虽然这个参数并不是根据地N+I帧数据计算出来的，但是由于相邻帧之间有很大的连续性，所以可以认为它们计算出来的全局变量是相同的，这样就可以实现正确并且实时的处理了。这种方法同样适用于AWB、3DNR、HDR模块。
    
      因此，当前帧显示的直方图以及亮度信息均是统计于前一帧的直方图和亮度信息，当前帧根据这些统计信息再进行AE策略的调整。



当曝光误差超过容许值需要调整时，算法需要计算两个值，即

当前帧的曝光量，由sensor 曝光时间、sensor 增益、ISP 增益组成。需要注意的是，sensor 的曝光时间和增益通常是非连续的，很可能与AE算法输出的目标参数并不相同，所以当前曝光参数的准确值需要通过sensor 驱动从sensor 寄存器中直接读取，而不能使用AE算法缓存的目标值。
增益系数，g=target/measured, 其中target 为理想画面亮度, measured 为当前画面亮度的实测值。 由于sensor 的本质是一个线性元件，若暂不考虑像素饱和等非线性因素，只要在当前曝光总量的基础上乘以系数g，就可以使画面目标亮度达到理想值。
因此AE 算法的核心任务就是计算正确的g参数，这个参数能够使画面得到正确的曝光。

当计算出正确的亮度参数后，一般并不会让其立刻在下一帧图像就生效。这是因为如果增益变化较大，图像就会产生闪烁，主观感受不好。通常人们更喜欢画面平滑过渡，因此每帧图像的增益变化不宜过大。实现平滑的方法就是给新的参数人为施加一个阻尼，使其缓慢地向新参数过渡。用数学公式描述就是

g(n)= (1-s) * g(n-1) +s * g_target

不妨取 s=0.2，此时每个g参数包含80%的旧参数和20%的目标参数，经过若干帧后旧参数自然衰减，新参数收敛到目标参数，即 g(n)=g_target

从数学上看 (1-0.2)^10=0.1, (1-0.2)^30=0.001, 说明10帧之后（约0.3秒）旧参数的比重下降到10%，30帧之后（约1秒）旧参数的比重可忽略。对于典型的安防应用场景，一般建议经过8~16帧图像过渡到理想亮度。而对于运动和车载型应用，由于场景动态变化大且快，一般建议经过3~4帧图像过渡到理想亮度。

具体计算：

假设g(0) = 20; g_garget=45,则由阻尼公式可知：

n=1时，g(1) = 0.8*20+0.2*45;

n = 2时，g(2)=0.8*g(1)+0.2*45=

n=3时，g(3)=0.8*g(2)+0.2*45 = 

n = 4时，g(4) = 0.8*g(3)+0.2*45 = 



当n=10时，g(10) = 

参数分解

当根据路径规划策略计算出下一帧的g参数后，需要遵循一定的策略和约束把g参数进一步映射为sensor 曝光时间、sensor 增益、ISP 增益等设备控制参数。如前所述，曝光时间可以提高图像信噪比，所以在约束边界内应尽可能先将曝光时间用满，然后依照sensor 的硬件约束分配sensor 增益，最后将剩余的增益全部分配给ISP 数字增益。

5. 参数同步

前面分解出来的控制参数必须同步生效才能使画面获得预期的曝光。如果某一项参数未能与其他几项同步生效，则画面会因为短暂过亮、过暗等原因出现闪烁，这是需要避免的。

另一方面，所有参数都需要在一个特定的时间窗口内生效，即前一帧图像已经结束，新一帧图像尚未开始的这段时间，也就是sensor的垂直消隐（vertical blanking）窗口，这个窗口时间很短，典型值在3~5毫秒左右，更短的可以到1ms，如下图所示。



目前主流的sensor都是使用I2C总线进行寄存器读写，而I2C总线的最大时钟频率是400kHz，读写一个16bit寄存器差不多每次需要0.1ms，而完成一帧图像的相关配置常常需要10~20次以上读写，所以在垂直消隐区完成sensor 寄存器配置时间压力是很大的。而且这还是单纯的配置参数，并不考虑3A算法本身所需的计算时间。实际上，在常见的软件硬件架构中，就是把全部消隐时间都分配给3A算法往往都是不够用的。

如果配置sensor 增益时错过了这个窗口，新一帧图像已经开始，则画面的亮度就会在一帧中间发生变化，上半部分使用旧的参数，下半部分使用新的参数，这种情况也是闪烁的一种，是需要避免的。

现在的sensor 为了方便使用，缓解配置参数时间窗口过短的压力，往往都支持一组影子（shadow）寄存器，需要同步生效的参数（曝光时间和增益）可以在任何时间点写入shadow寄存器，当sensor开始捕捉新的一帧图像之前，会自动把shadow 寄存器的内容同步到实际生效的寄存器，这样就把几个毫秒的时间窗口扩展成一帧时间，极大地缓解了用户压力。



需要注意的是，虽然这个方案为软件争取到了一帧的缓冲时间，但同时也意味着系统的响应延迟（latency）增加了一帧，即根据第N帧统计数据生成的新控制参数只能在第N+2帧才开始生效，因为软件需要第在N+1帧时间内完成算法的计算工作。同理，根据第N+2帧统计生成的新控制参数需要在第N+4帧才开始生效，以此类推。因此，如果环境光照条件在第N+1帧发生剧变，算法会在第N+2帧结束时检测到画面异常，在第N+3帧中计算出新的参数，在第N+4帧中实际进行补偿。

事实上，如果CPU的任务比较繁忙，或者每帧的时间很短，则一帧的时间可能还不一定够3A算法完成所有计算，此时则需要考虑继续增加一帧的缓冲时间。
