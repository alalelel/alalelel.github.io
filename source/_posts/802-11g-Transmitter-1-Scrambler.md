---
title: 📡 802.11g 发射机 | 1. 扰码
date: 2019-11-01 22:46:32
tags:
---
扰码器（Sreambler）是什么？扰码是本质上是将需要传输的数据进行随机化处理，通过把输入数据“耦合”到一个随机序列上来实现，而序列的随机性能就保证通信性能的平稳性。
<!-- more -->

802.11-2007 **17.3.5.4 PLCP DATA scrambler and descrambler**规定DATA field应该使用一个长127的帧同步扰码器进行扰码，该扰码器使用生成如下生成多项式：

$$S(x)=x^7+x^4+1$$

这个生成多项式对应了一个反馈移位寄存器结构：

![Figure-17-7-Data-scrambler](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/Figure-17-7-Data-scrambler)

左侧的反馈移位寄存器构成了一个周期为127的随机序列发生器，其发生序列只与移位寄存器的初始状态有关，在初始状态全0的状态下其输出序列为：
```
00001110 11110010 11001001 00000010 00100110 00101110
10110110 00001100 11010100 11100111 10110100 00101010 
11111010 01010001 10111000 1111111
```
随机序列与输入序列进行异或运算得出扰码序列，在接收端使用同样的随机序列与扰码序列异或即可恢复原输入序列，这里利用到的原理是：
$$
\begin{split}
Trans(x) = & Input(x) \oplus S(x) \\\\
Recv(x) = & Trans(x) \oplus S(x) \\\\
= & Input(x) \oplus S(x) \oplus S(x) \\\\
= & Input(x) \oplus 0 \\\\
= & Input(x)
\end{split}
$$

由于随机序列只与移位寄存器的初始状态有关，只需要发射机与接收机约定使用共同的初始值，并且在每次发射前进行重置即可保证加扰序列可以在接收端进行解扰，802.11规定该初始值为1011101。

# Verilog实现
``` verilog
module stream_scramble(
    input rst_n,
    input clk,
    input din,
    input en,
    input [7:1] seed,
    input start_frame,
    input [11:0] sig_length, // used for zeroing tail bits
    output reg dout,
    output reg data_valid
    );

    reg [7:1] scrambler;
    reg [12:0] bit_counter;
    always @ (posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            dout <= 0;
            data_valid <= 0;
            scrambler <= 0;
            bit_counter <= 0;
        end
        else begin
            if (start_frame) begin
                scrambler <= seed;
                bit_counter <= 0;
            end
            else begin
                if (en) begin
                    // zeroing tail bits
                    dout <= (bit_counter >= sig_length * 8 + 16 && bit_counter < sig_length * 8 + 22) ? 0 : 
                            din + scrambler[7] + scrambler[4];
                    data_valid <= 1;
                    scrambler <= {scrambler[6:1], scrambler[7] + scrambler[4]};
                    bit_counter <= bit_counter + 1;
                end
                else begin
                    dout <= 0;
                    data_valid <= 0;
                end
            end
        end
    end
endmodule
```
#### 仿真
将扰码器置于全1的初始值，在全0输入的情况下观察扰码器的输出，其序列应该满足：
```
00001110 11110010 11001001 00000010 00100110 00101110
10110110 00001100 11010100 11100111 10110100 00101010 
11111010 01010001 10111000 1111111
```
![scrambler-isim-2](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/scrambler-isim-2)
对比发现仿真与要求相吻合。

#### 基带数据与扰码器输出
![baseband_bits](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/baseband_bits)
#### 没有扰码器时
![baseband_time_domin_without_scrambler](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/baseband_time_domin_without_scrambler)
#### 添加扰码器后
![baseband_time_domain_with_scrambler](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/baseband_time_domain_with_scrambler)

在基带信号中，可见存在大量的连续的0和1，当没有扰码器时，产生的时域信号存在非常明显的周期性，且出现了大量的峰值，EVM恶化。在添加扰码器后情况明显改善。