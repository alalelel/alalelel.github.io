---
title: 📡802.11g Transmitter|Scrambler
date: 2019-11-09 9:33:09
tags:
---
<font face="微软雅黑" size=4>**📍扰码器简介**</font>
<!--more -->
<table><tr><td bgcolor=#F0F8FF>
&#8195;&#8195;数字通信中，如果经常出现长的0或1序列，将会影响位同
步的建立和保持。在发射机中使用扰码，可以<font color=red>避免这种数据对接收机定时的不利影响</font>。同时，为了限制电路中存在的不同程度的非线性特性对其他电路通信造成的串扰，<font color=red>要求数字信号的最小周期足够长</font>。将数字信号变换成<font color=red>具有近似于白噪声统计特性的数字序列</font>即可满足要求，这通常用加扰来实现。</td></tr></table>

<font face="黑体微软雅黑" size=4>**🎈原理**</font>

>&#8195;&#8195;所谓加扰，就是**不用增加冗余而扰乱信号**，改变数字信号统计特性，使其具有近似白噪声统计特性的一种技术。在OFDM中，DATA域数据的处理，首先需要进行加扰操作。整个DATA域数据使用一个长度为127的帧同步扰码器加扰。8位PSDU数据帧转换成**串行比特流**，其中，LSB在前，MSB在最后。扰码器使用下列生成多项式:$$\mathbf{S(x) = x^7 + x^4 + 1}$$也就是将数据数据送入该多项式，然后输出。输出后的结果就是扰码之后的结果。该扰码器的最高幂是7，首先要定义一个7位非零的伪随机处置状态，比如说是 1011101。经过上面的多项式之后，生成一个127位的随机数。

<font face="黑体微软雅黑" size=4>**🎈实现**</font>
&#8195;&#8195;扰码器可以用一个7位移位寄存器SCRAMBLER实现。扰码模块的模块框图和端口定义如下：
![Picture1.png](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/fimg/Picture1.png)

| 端口名 | 位宽 | 输入/输出|说明|
| :------: | :------: | :------: |:------: |
| SCRAM_DIN | 1 | Input |扰码器输入信号
| SCRAM_ND | 1 | Input |扰码器输入有效，与输入信号同步拉高
| SCRAM_LOAD| 1 | Input |扰码器初始设置信号
| SCRAM_RST | 1 | Input |复位信号，低电平有效
| SCRAM_CLK | 1 | Input |时钟信号(60MHz)
| SCRAM_SEED | 7 | Input |扰码器初始状态值，此处为7'b1011101
| SCRAM_DOUT | 1 | Output |扰码器输出信号
| SCRAM_RDY | 1 | Output |扰码器输出有效，与输出信号同步拉高

<font face="黑体微软雅黑" size=4>**🎈代码**</font>
&#8195;&#8195;当扰码器初始设置信号SCRAM_LOAD为高电平时，扰码器被设置成一个非零的伪随机状态。此处取非零伪随机状态为SCRAM_SEED=1011101
``` python
SCRAM_SEED = 7'b1011101;
if(SCRAM_LOAD)
    SCRAMBLER <= SCRAM_SEED;
```
&#8195;&#8195;当扰码器输入有效信号SCRAM_ND为高电平时，每个时钟周期扰码器状态左移一位，并且第7位和第4位进行异或运算，所得的值一方面作为寄存器的输入，另一方面和输入的串行比特流进行异或运算得到加扰后的输出数据SCRAM_DOUT。
``` python
if(SCRAM_ND)
    begin
        SCRAM_DOUT <= SCRAM_DIN + data_temp[7] + SCRAMBLER[4];
        SCRAMBLER <= {SCRAMBLER[6:1],data_temp[7] + SCRAMBLER[4]};
    end
```
<font face="黑体微软雅黑" size=4>**🎈仿真结果**</font>
&#8195;&#8195;对工程文件进行综合、布局、布线后仿真，得到如下图所示的仿真结果:
![Picture2.png](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/fimg/Picture2.png)
&#8195;&#8195;上图中，din是输入扰码器的两个Symbol，每个Symbol为144个bit;dout为扰码处理后的输出，同样输出两个Symbol的数据，每个Symbol有144个bit已扰码数据。
>&#8195;&#8195;将仿真通过的工程文件使用ChipScope添加观察信号采样时钟、触发信号和待观察信号后重新综合、布局布线生成bit文件，下载到目标板后用ChipScope进行在线测试，得到观测结果如下图所示。下图中的DIN对应上图中的din，下图中的DOUT对应上图中的dout。下图的在线测试结果与上图的后仿真结果吻合，验证了设计的正确性。

![Picture3.png](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/fimg/Picture3.png)


