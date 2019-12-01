---
title: 📡 802.11g 发射机 | 7.加窗
date: 2019-11-07 17:12:13
tags:
---
#### Windowing Port defination
``` verilog
module windowing_combiner(
    input rst_n,
    input clk_sys,
    input start_frame,
    input end_frame,
    // Training Seq Input
    input signed [7:0] ts_re,
    input signed [7:0] ts_im,
    input ts_en,
    input ts_last,
    // IFFT Input
    input signed [7:0] ifft_re,
    input signed [7:0] ifft_im,
    input ifft_en,
    input [5:0] ifft_index,
    // Tx Output
    output reg signed [7:0] tx_re,
    output reg signed [7:0] tx_im,
    output reg tx_valid
    );
```

#### Windowing
``` verilog
always @(*) begin
    save_first_val = 0;
    case (state)
        IDLE : begin
            mux_out_re = 0;
            mux_out_im = 0;
        end
        MUX_TS : begin
            if (ts_last) begin
                mux_out_re = ts_re + (ifft_re >>> 1);
                mux_out_im = ts_im + (ifft_im >>> 1);
            end
            else begin
                mux_out_re = ts_re;
                mux_out_im = ts_im;
            end
        end
        MUX_IFFT : begin
            if (last_ifft_index == 63 && ifft_index == 0) begin
                save_first_val = 1;
                mux_out_re = ifft_re;
                mux_out_im = ifft_im;
            end
            else if (last_ifft_index == 63 && ifft_index == 48) begin
                mux_out_re = (ifft_re >>> 1) + first_ifft_re;
                mux_out_im = (ifft_im >>> 1) + first_ifft_im;
            end
            else begin
                mux_out_re = ifft_re;
                mux_out_im = ifft_im;
            end
        end
        default: begin
            mux_out_re = 0;
            mux_out_im = 0;
        end
    endcase
end
// Save Value
always @(posedge clk_sys or negedge rst_n) begin
    if (!rst_n) begin
        first_ifft_re <= 0;
        first_ifft_im <= 0;
    end
    else begin
        if (save_first_val) begin
            first_ifft_re <= mux_out_re >>> 1;
            first_ifft_im <= mux_out_im >>> 1;
        end
    end
end
// Output
always @(posedge clk_sys or negedge rst_n) begin
    if (!rst_n) begin
        tx_re <= 0;
        tx_im <= 0;
        tx_valid <= 0;
    end
    else begin
        tx_re <= mux_out_re;
        tx_im <= mux_out_im;
        tx_valid <= ifft_en | ts_en;
    end
end
```

#### Windowing Simulation
不用第一张
![window-combiner-modelsim-1](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/window-combiner-modelsim-1)

第二张:
![window-combiner-modelsim-2](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/window-combiner-modelsim-2)
可见加窗模块将训练序列与后续OFDM符号序列合并在一起,且每个Symbol均储存第一个值,用于加窗(把左边裁一下)