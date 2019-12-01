---
title: 📡 802.11g 发射机 | 4.星座映射
date: 2019-11-04 17:10:27
tags:
---
#### Constellation Port defination
``` verilog
module stream_constellate(
    input rst_n,
    input clk,
    input din,
    input en,
    input [3:0] sig_rate,
    output reg [7:0] re,
    output reg [7:0] im,
    output reg [5:0] index,
    output reg data_valid
    );
```
#### Constellation 
``` verilog
// constellation map
reg map_valid;
reg [7:0] re_temp;
reg [7:0] im_temp;
always @ (posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        re_temp <= 0;
        im_temp <= 0;
        map_valid <= 0;
    end
    else begin
        if (buf_data_valid) begin
            case (sig_rate)
                // BPSK
                4'b1101, 4'b1111: begin
                    // Scaler=1
                    case (pre_map_buffer[0])
                        1'b0: re_temp <= 8'b11000000; // -1
                        1'b1: re_temp <= 8'b01000000; // 1
                    endcase
                    im_temp <= 0;
                    map_valid <= 1;
                end
                // QPSK
                4'b0101, 4'b0111: begin
                    // Scaler=sqrt(2)
                    case (pre_map_buffer[0])
                        1'b0: re_temp <= 8'b11010011; // -1
                        1'b1: re_temp <= 8'b00101101; // 1
                    endcase
                    case (pre_map_buffer[1])
                        1'b0: im_temp <= 8'b11010011; // -1
                        1'b1: im_temp <= 8'b00101101; // 1
                    endcase
                    map_valid <= 1;
                end
                // 16QAM
                4'b1001, 4'b1011: begin
                    // Scaler=sqrt(10)
                    case (pre_map_buffer[1:0])
                        2'b00: re_temp <= 8'b11000011; // -3
                        2'b10: re_temp <= 8'b11101100; // -1
                        2'b11: re_temp <= 8'b00010100; // 1
                        2'b01: re_temp <= 8'b00111101; // 3
                    endcase
                    case (pre_map_buffer[3:2])
                        2'b00: im_temp <= 8'b11000011; // -3
                        2'b10: im_temp <= 8'b11101100; // -1
                        2'b11: im_temp <= 8'b00010100; // 1
                        2'b01: im_temp <= 8'b00111101; // 3
                    endcase
                    map_valid <= 1;
                end
                // 64QAM
                4'b0001, 4'b0011: begin
                    // Scaler=sqrt(42)
                    case (pre_map_buffer[2:0])
                        3'b000: re_temp <= 8'b10111011; // -7
                        3'b001: re_temp <= 8'b11001111; // -5
                        3'b011: re_temp <= 8'b11100010; // -3
                        3'b010: re_temp <= 8'b11110110; // -1
                        3'b110: re_temp <= 8'b00001010; // 1
                        3'b111: re_temp <= 8'b00011110; // 3
                        3'b101: re_temp <= 8'b00110001; // 5
                        3'b100: re_temp <= 8'b01000101; // 7
                    endcase
                    case (pre_map_buffer[5:3])
                        3'b000: im_temp <= 8'b10111011; // -7
                        3'b001: im_temp <= 8'b11001111; // -5
                        3'b011: im_temp <= 8'b11100010; // -3
                        3'b010: im_temp <= 8'b11110110; // -1
                        3'b110: im_temp <= 8'b00001010; // 1
                        3'b111: im_temp <= 8'b00011110; // 3
                        3'b101: im_temp <= 8'b00110001; // 5
                        3'b100: im_temp <= 8'b01000101; // 7
                    endcase
                    map_valid <= 1;
                end
                // Invalid SIG_RATE input!
                default: begin
                    re_temp <= 0;
                    im_temp <= 0;
                    map_valid <= 0;
                end
            endcase
        end
        else begin
            map_valid <= 0;
            re_temp <= 0;
            im_temp <= 0;
        end
    end
end
```
BPSK modulation simulation
![bpsk-mod-isim-1](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/bpsk-mod-isim-1)

QPSK
![qpsk-mod-isim-1](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/qpsk-mod-isim-1)

16QAM
![16qam-mod-isim-1](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/16qam-mod-isim-1)

64QAM
![64qam-mod-isim-1](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/64qam-mod-isim-1)