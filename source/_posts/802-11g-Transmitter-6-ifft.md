---
title: 📡 802.11g 发射机 | 6.IFFT
date: 2019-11-06 17:12:03
tags:
---
#### IFFF Port defination
``` verilog 
module stream_ifft(
    input rst_n,
    input clk_din,
    input [7:0] din_re,
    input [7:0] din_im,
    input din_valid,
    output [7:0] dout_re,
    output [7:0] dout_im,
    output [5:0] dout_index,
    output dout_valid
    );
```
#### IFFT IP core and its configuration
对IP核进行基本的配置，包括缩放系数、转换方向和循环前缀长度
``` verilog
// FFT core
xfft_0 IFFT (
    .aclk(clk_din),                                                // input wire aclk
    .aresetn(rst_n),                                          // input wire aresetn
    
    .s_axis_config_tdata(s_axis_config_tdata),                  // input wire [15 : 0] s_axis_config_tdata
    .s_axis_config_tvalid(s_axis_config_tvalid),                // input wire s_axis_config_tvalid
    .s_axis_config_tready(s_axis_config_tready),                // output wire s_axis_config_tready
    
    .s_axis_data_tdata(s_axis_data_tdata),                      // input wire [31 : 0] s_axis_data_tdata
    .s_axis_data_tvalid(s_axis_data_tvalid),                    // input wire s_axis_data_tvalid
    .s_axis_data_tready(s_axis_data_tready),                    // output wire s_axis_data_tready
    .s_axis_data_tlast(s_axis_data_tlast),                      // input wire s_axis_data_tlast
    
    // Transform result
    .m_axis_data_tdata(m_axis_data_tdata),                      // output wire [31 : 0] m_axis_data_tdata
    // Index output
    .m_axis_data_tuser(m_axis_data_tuser),                      // output wire [15 : 0] m_axis_data_tuser
    .m_axis_data_tvalid(m_axis_data_tvalid),                    // output wire m_axis_data_tvalid
    .m_axis_data_tready(m_axis_data_tready),                    // input wire m_axis_data_tready
    .m_axis_data_tlast(m_axis_data_tlast),                      // output wire m_axis_data_tlast
    
    // Overflow AXI (not used)
    .m_axis_status_tdata(m_axis_status_tdata),                  // output wire [7 : 0] m_axis_status_tdata
    .m_axis_status_tvalid(m_axis_status_tvalid),                // output wire m_axis_status_tvalid
    .m_axis_status_tready(m_axis_status_tready),                // input wire m_axis_status_tready
    
    // Events
    .event_frame_started(event_frame_started),                  // output wire event_frame_started
    .event_tlast_unexpected(event_tlast_unexpected),            // output wire event_tlast_unexpected
    .event_tlast_missing(event_tlast_missing),                  // output wire event_tlast_missing
    .event_fft_overflow(event_fft_overflow),                    // output wire event_fft_overflow
    .event_status_channel_halt(event_status_channel_halt),      // output wire event_status_channel_halt
    .event_data_in_channel_halt(event_data_in_channel_halt),    // output wire event_data_in_channel_halt
    .event_data_out_channel_halt(event_data_out_channel_halt)  // output wire event_data_out_channel_halt
    );
// configuration
always @(posedge clk_din or negedge rst_n) begin
    if (!rst_n) begin
        s_axis_config_tvalid <= 0;
        s_axis_config_tdata <= {1'b0, 6'b101110, 1'b0, 2'b00, 6'b010000}; // SCALE_SCH=10 11 10 IFFT CP_LEN=16
    end
    else if (s_axis_config_tready) begin
        s_axis_config_tvalid <= 1;
    end
    else begin
        s_axis_config_tvalid <= 0;
    end
end
```
#### IFFT Simulation
![ifft-modelsim-part-1](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/ifft-modelsim-part-1)

可见输出包括了循环前缀（IP核功能），且存在一定的时延