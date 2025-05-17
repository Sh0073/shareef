
`timescale 1ns / 1ps

module fir_r2l_dynamic_efficient (
input  wire        clk,
input  wire        rst,
input  wire  [7:0]  x_in,        // 8-bit signed input
input  wire [0:11]        h_mask,      // 12-bit coefficient mask (1 for 2^-i)
output reg   [15:0] y_out        // Wider output for multiple shifted adds
);


// Internal logic
reg  [15:0] acc;
integer i;

always @(posedge clk or posedge rst) begin
    if (rst) begin
        y_out <= 0;
        acc   <= 0;
    end else begin
        acc = 0;

        // Accumulate shifted versions of x_in from MSB to LSB
        for (i = 11; i >= 0; i = i - 1) begin
            if (h_mask[i]) begin
                acc = acc + (x_in >>> i);  // signed arithmetic shift
            end
        end

        y_out <= acc;
    end
end

endmodule
`timescale 1ns / 1ps

module fir_r2l_dynamic_efficient_tb;

    // Inputs
    reg clk;
    reg rst;
    reg [7:0] x_in;
    reg [11:0] h_mask;

    // Output
    wire [15:0] y_out;

    // Instantiate the DUT
    fir_r2l_dynamic_efficient uut (
        .clk(clk),
        .rst(rst),
        .x_in(x_in),
        .h_mask(h_mask),
        .y_out(y_out)
    );

    // VCD Dump
    initial begin
        $dumpfile("fir_r2l_dynamic_efficient.vcd");
        $dumpvars(0, fir_r2l_dynamic_efficient_tb);
    end

    // Clock generation
    initial clk = 0;
    always #5 clk = ~clk;

    // Stimulus
    initial begin
        $display("Time\tclk\trst\tx_in\t\th_mask\t\ty_out");
        $monitor("%0t\t%b\t%b\t%8b\t%12b\t%6d", $time, clk, rst, x_in, h_mask, y_out);

        // Initial reset
        rst = 1;
        x_in = 0;
        h_mask = 0;
        #12;

        rst = 0;

        // Test 1
        x_in = 8'b10101010;              // 170
        h_mask = 12'b001010101010;
        #40;

        // Test 2
        x_in = 8'b00001111;              // 15
        h_mask = 12'b001010101010;
        #40;

        // Test 3
        x_in = 8'b11110000;              // 240 (if unsigned)
        h_mask = 12'b001010101010;
        #40;

        $finish;
    end

endmodule


// uart 


module baud_gen(
    input clk,
    input reset,
    output reg baud_en
);
    parameter CLK_FREQ = 50000000;  // 50 MHz
    parameter BAUD_RATE = 9600;     // 9600 bps

    localparam integer BAUD_COUNT_MAX = CLK_FREQ / BAUD_RATE;
    reg [31:0] counter;

    always @(posedge clk or posedge reset) begin
        if (reset) begin
            counter <= 0;
            baud_en <= 0;
        end else begin
            if (counter == BAUD_COUNT_MAX/2 - 1) begin
                baud_en <= 1;
                counter <= 0;
            end else begin
                counter <= counter + 1;
                baud_en <= 0;
            end
        end
    end
endmodule   



module uart_tx(
    input clk,
    input reset,
    input baud_en,
    input [7:0] t_byte,
    input load,
    output reg tx,
    output reg ready
);

    reg [9:0] shift_reg;
    reg [3:0] bit_cnt;
    reg sending;

    always @(posedge clk or posedge reset) begin
        if (reset) begin
            tx <= 1'b1;
            shift_reg <= 10'b1111111111;
            bit_cnt <= 0;
            sending <= 0;
            ready <= 1;
        end else begin
            if (load && ready) begin
                shift_reg <= {1'b1, t_byte, 1'b0}; // {stop bit, 8 data bits, start bit}
                sending <= 1;
                bit_cnt <= 0;
                ready <= 0;
            end else if (sending && baud_en) begin
                tx <= shift_reg[0];
                shift_reg <= shift_reg >> 1;
                bit_cnt <= bit_cnt + 1;
                if (bit_cnt == 9) begin
                    sending <= 0;
                    ready <= 1;
                end
            end
        end
    end
endmodule   




module uart_rx(
    input clk,
    input reset,
    input baud_en,
    input serial_in,
    output reg [7:0] r_byte,
    output reg done
);

    reg [9:0] shift_reg;
    reg [3:0] bit_cnt;
    reg receiving;
    reg half_bit_sample;

    always @(posedge clk or posedge reset) begin
        if (reset) begin
            shift_reg <= 10'b0;
            bit_cnt <= 0;
            receiving <= 0;
            half_bit_sample <= 0;
            r_byte <= 8'b0;
            done <= 0;
        end else begin
            done <= 0;

            if (!receiving) begin
                if (!serial_in) begin  // Start bit detected
                    receiving <= 1;
                    half_bit_sample <= 1;
                    bit_cnt <= 0;
                end
            end else if (baud_en) begin
                if (half_bit_sample) begin
                    // Skip half bit time
                    half_bit_sample <= 0;
                end else begin
                    shift_reg <= {serial_in, shift_reg[9:1]};
                    bit_cnt <= bit_cnt + 1;
                    if (bit_cnt == 9) begin
                        receiving <= 0;
                        r_byte <= shift_reg[8:1]; // Get only data bits
                        done <= 1;
                    end
                end
            end
        end
    end
endmodule   


module uart_top(
    input clk,
    input reset,
    input [7:0] data_in,
    input load_tx,
    output tx,
    output [7:0] data_out,
    output done,
    output ready
);

    wire baud_en;
    wire internal_rx;

    baud_gen #(
        .CLK_FREQ(50000000),
        .BAUD_RATE(9600)
    ) baud_gen_inst (
        .clk(clk),
        .reset(reset),
        .baud_en(baud_en)
    );

    uart_tx tx_inst (
        .clk(clk),
        .reset(reset),
        .baud_en(baud_en),
        .t_byte(data_in),
        .load(load_tx),
        .tx(tx),
        .ready(ready)
    );

    assign internal_rx = tx;  // Loopback

    uart_rx rx_inst (
        .clk(clk),
        .reset(reset),
        .baud_en(baud_en),
        .serial_in(internal_rx),
        .r_byte(data_out),
        .done(done)
    );

endmodule
 







`timescale 1ns/1ps

module uart_tb;

    reg clk;
    reg reset;
    reg [7:0] data_in;
    reg load_tx;
    wire tx;
    wire [7:0] data_out;
    wire done;
    wire ready;

    uart_top uut (
        .clk(clk),
        .reset(reset),
        .data_in(data_in),
        .load_tx(load_tx),
        .tx(tx),
        .data_out(data_out),
        .done(done),
        .ready(ready)
    );

    always #10 clk = ~clk; // 50 MHz clock (20ns period)

    initial begin
        clk = 0;
        reset = 1;
        data_in = 8'b0;
        load_tx = 0;
        #100;

        reset = 0;
        #100;

        @(posedge clk);
        data_in = 8'b01010101; // First test
        load_tx = 1;
        @(posedge clk);
        load_tx = 0;

        wait(done);
        if (data_out == 8'b01010101)
            $display("Test 1 Passed: Received %b", data_out);
        else
            $display("Test 1 Failed: Received %b", data_out);

        #10000; // Gap

        @(posedge clk);
        data_in = 8'b10101010; // Second test
        load_tx = 1;
        @(posedge clk);
        load_tx = 0;

        wait(done);
        if (data_out == 8'b10101010)
            $display("Test 2 Passed: Received %b", data_out);
        else
            $display("Test 2 Failed: Received %b", data_out);

        #10000;

        $stop;
    end

endmodule


//traffic light


module traffic_light_controller (
    input wire clk, reset,
    input wire sound_sensor,
    input wire sensor_a1, sensor_a2, sensor_a3,
    input wire sensor_b1, sensor_b2, sensor_b3,
    input wire sensor_c1, sensor_c2, sensor_c3,
    input wire sensor_d1, sensor_d2, sensor_d3,
    input wire rc1, rc2, rc3, rc4,
    input wire sensor_ss1, sensor_ss2, sensor_ss3, sensor_ss4,  // Emergency vehicle sensors
    output reg light_t1_r, light_t1_g, light_t1_y,
    output reg light_t2_r, light_t2_g, light_t2_y,
    output reg light_t3_r, light_t3_g, light_t3_y,
    output reg light_t4_r, light_t4_g, light_t4_y,
    output reg cm  // camera trigger
);

// State encoding using localparams
localparam S0  = 5'd0,  S1  = 5'd1,  S2  = 5'd2,  S3  = 5'd3,  S4  = 5'd4,
           S5  = 5'd5,  S6  = 5'd6,  S7  = 5'd7,  S8  = 5'd8,  S9  = 5'd9,
           S10 = 5'd10, S11 = 5'd11, S12 = 5'd12, S13 = 5'd13, S14 = 5'd14,
           S15 = 5'd15, S16 = 5'd16, S17 = 5'd17, S18 = 5'd18, S19 = 5'd19,
           S20 = 5'd20, S21 = 5'd21, S22 = 5'd22, S23 = 5'd23, S24 = 5'd24,
           S25 = 5'd25, S26 = 5'd26, S27 = 5'd27, S28 = 5'd28, S29 = 5'd29,
           S30 = 5'd30, S31 = 5'd31, S32 = 5'd32, S33 = 5'd33, S34 = 5'd34;

// State variables
reg [4:0] current_state, next_state;

// Counter for timing
reg [31:0] counter;
localparam TIME_0S  = 32'd0, TIME_3S  = 32'd300_000_000, TIME_6S  = 32'd600_000_000, 
           TIME_12S = 32'd1_200_000_000, TIME_15S = 32'd1_500_000_000; 

// State transition and output logic
always @(posedge clk or posedge reset) begin
    if (reset) begin
        current_state <= S0;
        counter <= 0;
    end else begin
        if (counter == 0) begin
            current_state <= next_state;
            case (next_state)
                S0:  counter <= TIME_0S;  
                S1:  counter <= TIME_0S;  
                S2:  counter <= TIME_0S;  
                S3:  counter <= TIME_0S;
                S4:  counter <= TIME_0S;
                S5:  counter <= TIME_0S;
                S6:  counter <= TIME_6S;   
                S7:  counter <= TIME_12S;  
                S8:  counter <= TIME_15S;  
                S9:  counter <= TIME_3S;   
                S10: counter <= TIME_6S;   
                S11: counter <= TIME_12S;  
                S12: counter <= TIME_15S;  
                S13: counter <= TIME_3S;   
                S14: counter <= TIME_6S;   
                S15: counter <= TIME_12S;  
                S16: counter <= TIME_15S;  
                S17: counter <= TIME_3S;   
                S18: counter <= TIME_6S;   
                S19: counter <= TIME_12S;  
                S20: counter <= TIME_15S;  
                S21: counter <= TIME_3S;   
                S22: counter <= TIME_0S;
                S23: counter <= TIME_15S;   
                S24: counter <= TIME_15S;  
                S25: counter <= TIME_15S;  
                S26: counter <= TIME_15S;  
                S27: counter <= TIME_3S; 
                S28: counter <= TIME_3S;   
                S29: counter <= TIME_3S;   
                S30: counter <= TIME_3S;   
                S31: counter <= TIME_3S;   
                S32: counter <= TIME_3S;   
                S33: counter <= TIME_3S;   
                S34: counter <= TIME_3S;   
                default: counter <= TIME_0S;
            endcase
        end else begin
            counter <= counter - 1;
        end
    end
end

// State register
always @(posedge clk or posedge reset) begin
    if (reset)
        current_state <= S0;
    else
        current_state <= next_state;
end

// Next state logic
always @(*) begin
    case (current_state)
        S0: next_state = sound_sensor ? S22 : S1;
        S1: begin
            if (sensor_a1 | sensor_a2 | sensor_a3) next_state = S2;
            else if (sensor_b1 | sensor_b2 | sensor_b3) next_state = S3;
            else if (sensor_c1 | sensor_c2 | sensor_c3) next_state = S4;
            else if (sensor_d1 | sensor_d2 | sensor_d3) next_state = S5;
            else next_state = S0;
        end
        S2: begin
            if (sensor_a1) next_state = S6;
            else if (sensor_a2) next_state = S7;
            else if (sensor_a3) next_state = S8;
            else next_state = S2;
        end
        S3: begin
            if (sensor_b1) next_state = S10;
            else if (sensor_b2) next_state = S11;
            else if (sensor_b3) next_state = S12;
            else next_state = S3;
        end
        S4: begin
            if (sensor_c1) next_state = S14;
            else if (sensor_c2) next_state = S15;
            else if (sensor_c3) next_state = S16;
            else next_state = S4;
        end
        S5: begin
            if (sensor_d1) next_state = S18;
            else if (sensor_d2) next_state = S19;
            else if (sensor_d3) next_state = S20;
            else next_state = S5;
        end
        S6, S7, S8: next_state = S9;
        S9: next_state = S0;
        S10, S11, S12: next_state = S13;
        S13: next_state = S0;
        S14, S15, S16: next_state = S17;
        S17: next_state = S0;
        S18, S19, S20: next_state = S21;
        S21: next_state = S0;
        S22: begin
            if (!sound_sensor) next_state = S1;
            else if (sensor_ss1) next_state = S23;
            else if (sensor_ss2) next_state = S24;
            else if (sensor_ss3) next_state = S25;
            else if (sensor_ss4) next_state = S26;
            else next_state = S22;
        end
        S23: next_state = S27;
        S24: next_state = S28;
        S25: next_state = S29;
        S26: next_state = S30;
        S27: next_state = S31;
        S28: next_state = S32;
        S29: next_state = S33;
        S30: next_state = S34;
        S31: begin
            if (sensor_ss2) next_state = S24;
            else if (sensor_ss3) next_state = S25;
            else if (sensor_ss4) next_state = S26;
            else next_state = S0;
        end
        S32: begin
            if (sensor_ss1) next_state = S23;
            else if (sensor_ss3) next_state = S25;
            else if (sensor_ss4) next_state = S26;
            else next_state = S0;
        end
        S33: begin
            if (sensor_ss1) next_state = S23;
            else if (sensor_ss2) next_state = S24;
            else if (sensor_ss4) next_state = S26;
            else next_state = S0;
        end
        S34: begin
            if (sensor_ss1) next_state = S23;
            else if (sensor_ss2) next_state = S24;
            else if (sensor_ss3) next_state = S25;
            else next_state = S0;
        end
        default: next_state = S0;
    endcase
end

    // Output logic
   
// Output logic
always @(*) begin
    // Default: all lights red, camera off
    {light_t1_r, light_t1_y, light_t1_g} = 3'b100;
    {light_t2_r, light_t2_y, light_t2_g} = 3'b100;
    {light_t3_r, light_t3_y, light_t3_g} = 3'b100;
    {light_t4_r, light_t4_y, light_t4_g} = 3'b100;
    cm = 1'b0;

    case (current_state)
        S0, S1, S2, S3, S4, S5, S22: begin
            // All lights red
            {light_t1_r, light_t1_y, light_t1_g} = 3'b100;
            {light_t2_r, light_t2_y, light_t2_g} = 3'b100;
            {light_t3_r, light_t3_y, light_t3_g} = 3'b100;
            {light_t4_r, light_t4_y, light_t4_g} = 3'b100;
                cm = 1'b0;
        end

// T1 Green States
S6, S7, S8: begin
    {light_t1_r, light_t1_y, light_t1_g} = 3'b001; 

    // Display traffic status based on current state
    if (current_state == S6) 
        $display("light traffic(a1):100");
    else if (current_state == S7) 
        $display("medium traffic(a2):110");
    else if (current_state == S8) 
        $display("high traffic(a1):111");

    // Set camera signal based on road traffic detection (rc1)
    cm = rc1;  // Camera activated based on emergency signal or traffic condition

end
S9: begin
    {light_t1_r, light_t1_y, light_t1_g} = 3'b100;  // Yellow for T1
        cm = 1'b0;
end

// T1 Green States
S10, S11, S12: begin
    {light_t1_r, light_t1_y, light_t1_g} = 3'b001; 

    // Display traffic status based on current state
    if (current_state == S10) 
        $display("light traffic(a1):100");
    else if (current_state == S11) 
        $display("medium traffic(a2):110");
    else if (current_state == S12) 
        $display("high traffic(a1):111");

    // Set camera signal based on road traffic detection (rc1)
    cm = rc2;  // Camera activated based on emergency signal or traffic condition

end
S13: begin
    {light_t1_r, light_t1_y, light_t1_g} = 3'b100;  // Yellow for T1
        cm = 1'b0;
end


// T1 Green States
S14, S15, S16: begin
    {light_t1_r, light_t1_y, light_t1_g} = 3'b001; 

    // Display traffic status based on current state
    if (current_state == S14) 
        $display("light traffic(a1):100");
    else if (current_state == S15) 
        $display("medium traffic(a2):110");
    else if (current_state == S16) 
        $display("high traffic(a1):111");

    // Set camera signal based on road traffic detection (rc1)
    cm = rc3;  // Camera activated based on emergency signal or traffic condition

end
S17: begin
    {light_t1_r, light_t1_y, light_t1_g} = 3'b100;  // Yellow for T1
    cm = 1'b0;

end


// T1 Green States
S18, S19, S20: begin
    {light_t1_r, light_t1_y, light_t1_g} = 3'b001; 

    // Display traffic status based on current state
    if (current_state == S18) 
        $display("light traffic(a1):100");
    else if (current_state == S19) 
        $display("medium traffic(a2):110");
    else if (current_state == S20) 
        $display("high traffic(a1):111");

    // Set camera signal based on road traffic detection (rc1)
    cm = rc4;  // Camera activated based on emergency signal or traffic condition

end
S21: begin
    {light_t1_r, light_t1_y, light_t1_g} = 3'b100;  // Yellow for T1
    cm = 1'b0;

end



        // Emergency Green States
        S23: begin
            {light_t1_r, light_t1_y, light_t1_g} = 3'b001; // T1 green
            cm = rc1;
        end
        S24: begin
            {light_t2_r, light_t2_y, light_t2_g} = 3'b001; // T2 green
            cm = rc2;
        end
        S25: begin
            {light_t3_r, light_t3_y, light_t3_g} = 3'b001; // T3 green
            cm = rc3;
        end
        S26: begin
            {light_t4_r, light_t4_y, light_t4_g} = 3'b001; // T4 green
            cm = rc4;
        end

        // Emergency Yellow States
        S27: begin
            {light_t1_r, light_t1_y, light_t1_g} = 3'b010; // T1 yellow
            cm = 1'b0;
        end
        S28: begin
            {light_t2_r, light_t2_y, light_t2_g} = 3'b010; // T2 yellow
            cm = 1'b0;
        end
        S29: begin
            {light_t3_r, light_t3_y, light_t3_g} = 3'b010; // T3 yellow
            cm = 1'b0;
        end
        S30: begin
            {light_t4_r, light_t4_y, light_t4_g} = 3'b010; // T4 yellow
            cm = 1'b0;
        end

        // Emergency Transition States
        S31, S32, S33, S34: begin
            {light_t1_r, light_t1_y, light_t1_g} = 3'b100;
            {light_t2_r, light_t2_y, light_t2_g} = 3'b100;
            {light_t3_r, light_t3_y, light_t3_g} = 3'b100;
            {light_t4_r, light_t4_y, light_t4_g} = 3'b100;
            cm = 1'b0;
        end
    endcase
end



endmodule
