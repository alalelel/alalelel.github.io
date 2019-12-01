---
title: 📡 802.11g 发射机 | 3.交织
date: 2019-11-03 17:10:04
tags:
---
交织器输入输出端口
``` verilog
module stream_interleaver(
    input rst_n,
    input clk,
    input din,
    input [8:0] din_index,
    input en,
    input [3:0] sig_rate,
    output reg dout,
    output reg data_valid
    );
```
交织器代码(第一级交织写入地址生成)
``` verilog
// BPSK 48
4'b1101, 4'b1111: begin
    if (!write_addr_sel_1)
        write_addr_1 <= din_index[3:0] + (din_index[3:0] << 1) + din_index[5:4];
    else
        write_addr_1 <= din_index[3:0] + (din_index[3:0] << 1) + din_index[5:4] + 48;
    if (din_index == 47) begin
        write_addr_sel_1 <= ~write_addr_sel_1;
        read_en_1 <= 1;
    end
end
```
交织器代码(第二级交织写入地址生成)
``` verilog
// BPSK QPSK no need for 2nd itv
4'b1101, 4'b1111, 4'b0101, 4'b0111: begin
    write_addr_2 <= din_index_2;
    if (din_index_2 == 11 || din_index_2 == 23) begin
        read_en_2 <= 1;
    end
    if (din_index_2 < 23)
        din_index_2 <= din_index_2 + 1;
    else
        din_index_2 <= 0;
end
```
仿真结果
![interleaver-isim-1](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/interleaver-isim-1)

由于交织的特性，可见交织必须等数据全部写入后才可以读出，这里为发射机引入了时延