---
title: 📡 802.11g 发射机 | 8.基带信号发生控制器
date: 2019-11-08 17:58:34
tags:
---
#### Controller Port defination
``` verilog
module tx_controller(
    input global_rst_n,
    input clk_sys,
    // Control
    input start_tx,
    output reg busy,
    // Clock Gen
    input main_clk_locked,
    input clocks_locked,
    // Resets
    output reg reset_clocks,
    output reg reset_phy,
    // Scrambler
    output reg [7:1] scrambler_seed,
    // Config AXI IO
    input [31:0] s_config_tdata,
    input s_config_tvalid,
    // Tx Control
    output reg load_signal_field,
    input send_data_field,
    input different_rate,
    // Frame Control
    output reg start_frame,
    output reg start_ts,
    output reg start_sig,
    output reg start_data,
    output reg end_frame
    );
```

#### FSM
``` verilog
always @(*) begin
    next_state = state;
    case (state)
        // INIT Phase: reset hardware, wait loading config
        INIT0 : next_state = INIT1;
        INIT1 : next_state = INIT2;
        INIT2 : next_state = INIT3;
        INIT3 : if (s_config_tvalid) next_state = LOAD_SIG;
                else next_state = INIT3;
        LOAD_SIG : next_state = SET_CLK;
        SET_CLK : next_state = WAIT_CLK;
        WAIT_CLK : if (clocks_locked) next_state = PHY_RDY;
                    else next_state = WAIT_CLK;
        // TX Phase: wait tx command, then send payload
        PHY_RDY : if (start_tx) next_state = TX_HEADER;
                    else next_state = PHY_RDY;
        TX_HEADER : if (tx_header_counter >= 330) next_state = TX_DATA;
                    else next_state = TX_HEADER;
        TX_DATA: if (!send_data_field) next_state = TX_END;
                    else next_state = TX_DATA;
        TX_END : next_state = INIT3;
        default: next_state = INIT0;
    endcase
end
// output logic
always @(*) begin
    // reset_phy (active_low)
    case (state)
        INIT0, INIT1, INIT2: reset_phy = 0;
        default: reset_phy = 1;
    endcase
    // load_signal_field
    case (state)
        LOAD_SIG: load_signal_field = 1; 
        default: load_signal_field = 0;
    endcase
    // reset_clocks (active_high)
    case (state)
        SET_CLK: reset_clocks = (different_rate == 1);
        default: reset_clocks = 0;
    endcase
    // tx_header
    case (state)
        TX_HEADER: tx_header = 1;
        default:   tx_header = 0;
    endcase
    // end_frame
    case (state)
        TX_END:  end_frame = 1;
        default: end_frame = 0;
    endcase
    // busy
    case (state)
        PHY_RDY: busy = 0; 
        default: busy = 1;
    endcase
end
```
#### Timing Counters
``` verilog
// Transmitt Timing
// to clk_sys 20m
parameter START_TS_OFFSET = 32;
parameter START_TS_DURATION = 1;
// to clk_cb_10m 10m
parameter START_SIG_OFFSET = 1;
parameter START_SIG_DURATION = 2;
// to clk_cb 10m~90m
parameter START_DATA_OFFSET = 77;
parameter START_DATA_DURATION = 2;
always @(*) begin
    // start frame
    if (tx_header_counter < 4 && tx_header) start_frame = 1;
    else start_frame = 0;
    // traininig sequence
    if (tx_header_counter >= START_TS_OFFSET &&
        tx_header_counter < START_TS_OFFSET + START_TS_DURATION) start_ts = 1;
    else start_ts = 0;

    // signal field
    if (tx_header_counter >= START_SIG_OFFSET &&
        tx_header_counter < START_SIG_OFFSET + START_SIG_DURATION) start_sig = 1;
    else start_sig = 0;
    // data field
    if (tx_header_counter >= START_DATA_OFFSET &&
        tx_header_counter < START_DATA_OFFSET + START_DATA_DURATION) start_data = 1;
    else start_data = 0;

end

// header counter
always @(posedge clk_sys or negedge global_rst_n) begin
    if (!global_rst_n) begin
        tx_header_counter <= 0;
    end
    else begin
        if (tx_header) begin
            tx_header_counter <= tx_header_counter + 1;
        end
        else tx_header_counter <= 0;
    end
end
```

#### Controller Simulation
![tx-controller-modelsim-1](https://raw.githubusercontent.com/radonyl/pa_blog_img/master/img/tx-controller-modelsim-1)
发射机就是状态机+按时序产生信号