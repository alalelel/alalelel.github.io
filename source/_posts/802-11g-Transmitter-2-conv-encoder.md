---
title: 📡 802.11g 发射机 | 2.卷积编码
date: 2019-11-02 17:08:49
tags:
---
卷积编码器输入输出端口
``` verilog
module stream_conv_encode(input rst_n,
                          input start_frame,
                          input clk_din,
                          input clk_dout,
                          input [3:0] sig_rate,
                          input din,
                          input en,
                          output reg dout,
                          output reg [8:0] index,
                          output reg data_valid);

```

卷积编码器核心代码如下：
``` verilog
module conv_encoder(input rst_n,
                    input clk,
                    input din,
                    input en,
                    output reg [1:0] dout,
                    output reg data_valid);
    reg [6:1] shift_reg;
    always @ (posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            shift_reg  <= 6'd0;
            dout       <= 2'd0;
            data_valid <= 0;
        end
        else begin
            if (en) begin
                dout[0]    <= shift_reg[6] + shift_reg[5] + shift_reg[3] + shift_reg[2] + din;
                dout[1]    <= shift_reg[6] + shift_reg[3] + shift_reg[2] + shift_reg[1] + din;
                data_valid <= 1;
                shift_reg  <= {shift_reg[5:1], din};
            end
            else
                data_valid <= 0;
        end
    end
endmodule
```
#### 仿真图像

```
由IEEE802.11-2007附录G
// 输入序列为
24'b101100010011000000000000
// 输出序列应为
48'b110100011010000100000010001111100111000000000000
```
![conv-encoder-isim-1](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/conv-encoder-isim-1)

可见仿真与理论相吻合