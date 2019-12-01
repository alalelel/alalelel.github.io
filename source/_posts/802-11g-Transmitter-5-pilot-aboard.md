---
title: 📡 802.11g 发射机 | 5.插入导频
date: 2019-11-05 17:11:50
tags:
---
#### Pilot Aboard Port defination
``` verilog
module stream_pilot_aboard(
    input rst_n,
    input clk,
    input start_frame,
    input din_valid,
    input [7:0] din_re,
    input [7:0] din_im,
    input [5:0] din_index,
    output reg [7:0] dout_re,
    output reg [7:0] dout_im,
    output reg dout_valid
    );
```

#### OFDM Modulation
``` verilog
case (din_index_buf) // iFFT sub-carrier map
    0,1,2,3,4:
        write_addr[5:0] <= din_index_buf + 38;
    5,6,7,8,9,10,11,12,13,14,15,16,17:
        write_addr[5:0] <= din_index_buf + 39;
    18,19,20,21,22,23: 
        write_addr[5:0] <= din_index_buf + 40;
    24,25,26,27,28,29:
        write_addr[5:0] <= din_index_buf - 23; 
    30,31,32,33,34,35,36,37,38,39,40,41,42:
        write_addr[5:0] <= din_index_buf - 22;
    43,44,45,46,47 : 
        write_addr[5:0] <= din_index_buf - 21;
    default :
        write_addr[5:0] <= 0;
endcase
```

#### Pilot Insertion
``` verilog
case (pilot_postion)
    /*
        1 -1  1  1
       -1  1 -1 -1
    */
    2'b00: begin
        write_addr[5:0] <= 7;
        if (!pilot_polarity)
            ram_din_re <= 8'b01000000;
        else
            ram_din_re <= 8'b11000000;
        pilot_postion <= pilot_postion + 1;
    end
    2'b01: begin
        write_addr[5:0] <= 21;
        if (!pilot_polarity)
            ram_din_re <= 8'b11000000;
        else
            ram_din_re <= 8'b01000000;
        pilot_postion <= pilot_postion + 1;
    end
    2'b10: begin
        write_addr[5:0] <= 43;
        if (!pilot_polarity)
            ram_din_re <= 8'b01000000;
        else
            ram_din_re <= 8'b11000000;
        pilot_postion <= pilot_postion + 1;
    end
    2'b11: begin
        write_addr[5:0] <= 57;
        if (!pilot_polarity)
            ram_din_re <= 8'b01000000;
        else
            ram_din_re <= 8'b11000000;
        pilot_postion <= 0;
        pilot_aboard_start <= 0;
        read_en <= 1;
        write_addr_sel <= ~write_addr_sel;
    end
endcase
```

#### Simulation 
![pilot-aboard-isim-1](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/pilot-aboard-isim-1)