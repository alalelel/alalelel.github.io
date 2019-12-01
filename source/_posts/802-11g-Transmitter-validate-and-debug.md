---
title: 📡 802.11g 发射机 | 调试与测试
date: 2019-12-01 17:15:26
tags:
---
分模块设计完成后，需要进行组装和调试。按照框图将模块级联，接下来对模块的时序、数据完整性和数据正确性在仿真层面上进行验证。仿真验证通过后，将所有的模块打包成Tx80211自定义IP核，加入AD9361的IP核Block Design，进行接下来的板上验证。
![31512-draft-bd](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/31512-draft-bd)

# 发射机Modelsim仿真验证
首先使用matlab构造802.11g数据包，导出基带二进制流和生成好的基带信号。将matlab的基带二进制流储存在Tx_Ram中，并编写testbench，接下来在modelsim中进行基带数据生成器的全仿真。通过Modelsim观察并导出生成的波形，再将生成的波形与Matlab生成的标准数据进行比较。

#### Testbench
``` verilog
wire [23:0] SIGNAL;
reg [31:0] ram_addr = 0;
wire [7:0] DATA;
tx_ram TxRam(
    .addr(ram_addr),
    .SIGNAL(SIGNAL),
    .DATA(fifo_din)
);
initial
begin
    sim_idx = 0;
    # 40000; // wait clock to lock
    repeat(2) begin
        @(posedge clk)
        s_config_tdata[23:0] = SIGNAL;
        s_config_tvalid      = 1;
        for (idx = 0; idx < 66; idx = idx + 1) begin
            ram_addr   = idx;
            fifo_wr_en = 1;
            @(posedge clk);
        end
        fifo_wr_en = 0;
        while (busy) @(posedge clk);
        @(posedge clk);
        tx_now = 1;
        repeat(20) @(posedge clk);
        tx_now = 0;
        # 2000
        while (tx_data_valid) @(posedge clk);
        sim_idx = sim_idx + 1;
        # 10000;
    end
    $finish;
end
```

#### Tx Ram(节选)
用于储存发射二进制基带数据的Verilog文件，由matlab导出的二进制流数据生成。
``` verilog
module tx_ram(input [31:0] addr,
              output reg [23:0] SIGNAL,
              output reg [7:0] DATA);
    reg [7:0] data_ram [100:0];
    integer i;
    initial begin
        // zeroing ram
        for (i = 0; i < 101; i = i + 1)
            data_ram[i] = 0;
        // load data
        // generated at Sat Nov 23 17:41:48 2019 from file: E:\matlab_bb.txt
        SIGNAL = 24'b000000000000100000001011; // Rate:1101 Len:64
        data_ram[0] = 0'b00000000; // 0x00
        data_ram[1] = 0'b00000000; // 0x00
        data_ram[2] = 8'b10000000; // 0x80
        data_ram[3] = 8'b00000010; // 0x02
        // ...
        data_ram[62] = 8'b00111110; // 0x3e
        data_ram[63] = 8'b10101010; // 0xaa
        data_ram[64] = 8'b11101111; // 0xef
        data_ram[65] = 8'b10111100; // 0xbc
    end
    always @(*) begin
        DATA = data_ram[addr];
    end
endmodule
```

#### Modelsim 基带数据全仿真结果
设置modelsim捕捉各模块信号，按照扰码->卷积编码->交织->星座映射->添加导频->IFFT->加窗的顺序排列信号，观察信号传递时序是否正确，数据是否完整。
将
![bb-signal-gen-modelsim-total-1](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/bb-signal-gen-modelsim-total-1)



#### 基带信号发生器与Matlab标准数据对比
将数据导出后与matlab生成的标准发射信号进行对比，由于需要对比的数据量较多，使用Python编写了自动比较脚本，其输出结果如下：

![abs-diff-bb-vhdl-vs-matlab](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/abs-diff-bb-vhdl-vs-matlab)

结果显示：基带信号发生器产生的发射信号误差在允许范围内，且无错误。
# 发射机板上验证
在通过了Modelsim仿真验证后，将HDL工程综合后添加ILA核，添加相应的信号观察，布线后烧录在FPGA板上。此时Testbench实际上是ZYNQ7处理器中运行的程序，数据通过AXI总线写入Tx80211发射机IP核中的FIFO中，写入完毕后由ZYNQ7发出发射指令。通过设置ILA核触发信号，可定向捕捉所需信号，导出信号后与Matlab产生的标准信号进行对比。在上板调试中发现了之前仿真过程中没有发现的问题，详见下文。

在修正完所有Bug后，将最后的数据导出并与Matlab数据进行比较：

![abs-diff-bb-ila-vs-matlab](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/abs-diff-bb-ila-vs-matlab)

结果显示：基带信号发生器实际产生的发射信号误差在允许范围内，且无错误。

# 调试中遇到的问题
#### 1.卷积编码器帧同步置零问题
**问题现象**:
板上调试发现基带数据生成器中，卷积编码器输出的最初6个值与理论不符合，如图所示，其中红色为正确输出。

![problem-conv-encoder-not-reset](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/problem-conv-encoder-not-reset)

**问题分析**
经过对比，发现有且仅有最初6个数据与理论不符合，且稳定复现。考虑到卷积编码移位寄存器深度恰好等于6，逆推得到开始卷积前移位寄存器的值应为010110，恰好为上一次基带信号的最后6位。于是得出结论：卷积编码器在处理完一帧数据后没有重置，导致了问题。
**问题解决**
![code-conv-encoder-fix-1](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/code-conv-encoder-fix-1)
将卷积编码器的重置与帧开始信号同步，保证在处理每一帧数据前卷积编码器均处于全0状态。

#### 2.数据域Tail无需扰码需要置零的问题
**问题现象**
板上调试发现，输出的基带数据最后一个Symbol错误，而之前的所有symbol完全正确。
![problem-tail-not-zeroed](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/problem-tail-not-zeroed)
**问题分析**
仔细查看802.11协议，在附录G.1发现如下描述：
![zeroing-tail-bits-desc](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/zeroing-tail-bits-desc)
按照协议，在数据域最后6bit的Tail无需进行扰码，需要置零。而在基带信号生成器中没有考虑到这一点，导致最后一个Symbol发生错误。
**问题解决**
![tail-not-zeroed-fix-1](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/tail-not-zeroed-fix-1)
在扰码器模块中添加计数器，同时将计数器与SIGNAL Field的LENGTH进行比较，从而将相应的Tail bits置零。