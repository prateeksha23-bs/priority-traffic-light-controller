# priority-traffic-light-controller
Verilog-based priority traffic light controller using a Moore FSM on the Spartan-7 Boolean FPGA board, featuring NS/EW traffic phases, ambulance priority, timed yellow transitions, and all-red safety clearance.
# 🚦 Priority Traffic Light Controller using Moore FSM

A Verilog-based **Priority Traffic Light Controller** implemented using a Moore Finite State Machine (FSM) on the **Real Digital Boolean Spartan-7 FPGA board**. The project controls North-South (NS) and East-West (EW) traffic phases and supports ambulance priority while maintaining safe yellow and all-red transitions.

## 📌 Project Overview

The controller manages a four-way intersection using six FSM states. Ambulance detection inputs allow the controller to prioritize the requested direction without abruptly interrupting a yellow or all-red safety phase.

### ✨ Features

- Moore FSM-based traffic light control
- Six states for NS/EW traffic phases
- Green, yellow, and all-red safety intervals
- Ambulance priority for NS and EW directions
- Parameterized timing for simulation and hardware
- Verilog RTL design and testbench
- Functional verification using Vivado Simulator
- FPGA implementation targeting the Boolean Spartan-7 board

## 🖥️ FPGA Development Board

![Real Digital Boolean Spartan-7 FPGA Board](images/boolean-board.png)

**Board:** Real Digital Boolean FPGA Board  
**FPGA:** Xilinx Spartan-7 XC7S50

> Add your board photograph to the repository at `images/boolean-board.png`. Create the `images` folder if it does not exist.

## 🏗️ System Architecture

```text
       AMB_NS ─────┐
                   │
       AMB_EW ─────┤
                   ▼
             ┌───────────────┐
       clk ─►│               │
     reset ─►│   Moore FSM   │
             │   Controller  │
             └───────┬───────┘
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
    NS Traffic Lights     EW Traffic Lights
      RED/YELLOW/GREEN      RED/YELLOW/GREEN
```

## 🔄 FSM State Diagram

```text
          ┌───────────┐
          │ S0        │
          │ NS GREEN  │
          └─────┬─────┘
                ▼
          ┌───────────┐
          │ S1        │
          │ NS YELLOW │
          └─────┬─────┘
                ▼
          ┌───────────┐
          │ S2        │
          │ ALL RED   │
          └─────┬─────┘
                ▼
          ┌───────────┐
          │ S3        │
          │ EW GREEN  │
          └─────┬─────┘
                ▼
          ┌───────────┐
          │ S4        │
          │ EW YELLOW │
          └─────┬─────┘
                ▼
          ┌───────────┐
          │ S5        │
          │ ALL RED   │
          └─────┬─────┘
                │
                └──────────► S0
```

## 📋 State Table

| State | NS Light | EW Light | Purpose |
|---|---|---|---|
| S0 | Green | Red | NS traffic moves |
| S1 | Yellow | Red | NS transition |
| S2 | Red | Red | Clearance before EW |
| S3 | Red | Green | EW traffic moves |
| S4 | Red | Yellow | EW transition |
| S5 | Red | Red | Clearance before NS |

Normal sequence:

`S0 → S1 → S2 → S3 → S4 → S5 → S0`

## 🔌 Input and Output Signals

| Signal | Direction | Description |
|---|---|---|
| `clk` | Input | FPGA clock |
| `reset` | Input | Resets the controller |
| `AMB_NS` | Input | NS ambulance request |
| `AMB_EW` | Input | EW ambulance request |
| `NS_RED` | Output | NS red light |
| `NS_YELLOW` | Output | NS yellow light |
| `NS_GREEN` | Output | NS green light |
| `EW_RED` | Output | EW red light |
| `EW_YELLOW` | Output | EW yellow light |
| `EW_GREEN` | Output | EW green light |

## 💻 Verilog RTL

Save as `priority_traffic_light_controller.v`.

```verilog

`timescale 1ns / 1ps

module priority_traffic_light_controller #(
    parameter GREEN_TIME  = 500_000_000,
    parameter YELLOW_TIME = 200_000_000,
    parameter RED_TIME    = 100_000_000
)(
    input  wire clk,
    input  wire reset,
    input  wire AMB_NS,
    input  wire AMB_EW,

    output reg NS_RED,
    output reg NS_YELLOW,
    output reg NS_GREEN,
    output reg EW_RED,
    output reg EW_YELLOW,
    output reg EW_GREEN
);

    localparam S0_NS_GREEN  = 3'b000;
    localparam S1_NS_YELLOW = 3'b001;
    localparam S2_ALL_RED_1 = 3'b010;
    localparam S3_EW_GREEN  = 3'b011;
    localparam S4_EW_YELLOW = 3'b100;
    localparam S5_ALL_RED_2 = 3'b101;

    reg [2:0] current_state;
    reg [2:0] next_state;
    reg [31:0] timer;
    reg ns_request;
    reg ew_request;

    wire timer_done;

    assign timer_done =
    ((current_state == S0_NS_GREEN)  &&
     (timer >= GREEN_TIME - 1))  ||
     ((current_state == S1_NS_YELLOW) &&
      (timer >= YELLOW_TIME - 1)) ||
      ((current_state == S2_ALL_RED_1) &&
      (timer >= RED_TIME - 1))    ||
      ((current_state == S3_EW_GREEN)  &&
      (timer >= GREEN_TIME - 1))  ||
       ((current_state == S4_EW_YELLOW) &&
         (timer >= YELLOW_TIME - 1)) ||
        ((current_state == S5_ALL_RED_2) &&
         (timer >= RED_TIME - 1));

    always @(posedge clk or posedge reset) begin
        if (reset) begin
            ns_request <= 1'b0;
            ew_request <= 1'b0;
        end
        else begin
            if (AMB_NS)
                ns_request <= 1'b1;
            else if (current_state == S0_NS_GREEN)
                ns_request <= 1'b0;
            if (AMB_EW)
                ew_request <= 1'b1;
            else if (current_state == S3_EW_GREEN)
                ew_request <= 1'b0;

        end
    end

    always @(*) begin
        next_state = current_state;

        case (current_state)

            S0_NS_GREEN: begin
                if (AMB_NS || ns_request)
                    next_state = S0_NS_GREEN;
                else if (timer_done)
                    next_state = S1_NS_YELLOW;
            end

            S1_NS_YELLOW: begin
                if (timer_done)
                    next_state = S2_ALL_RED_1;
            end

            S2_ALL_RED_1: begin
                if (timer_done) begin
                if (ns_request || AMB_NS)
                        next_state = S0_NS_GREEN;
                    else
                        next_state = S3_EW_GREEN;
                end
            end

            S3_EW_GREEN: begin
                if (AMB_EW || ew_request)
                    next_state = S3_EW_GREEN;
                else if (timer_done)
                    next_state = S4_EW_YELLOW;
            end

            S4_EW_YELLOW: begin
                if (timer_done)
                    next_state = S5_ALL_RED_2;
            end

            S5_ALL_RED_2: begin
                if (timer_done) begin

                    if (ew_request || AMB_EW)
                        next_state = S3_EW_GREEN;
                    else
                        next_state = S0_NS_GREEN;
                end
            end

            default:
                next_state = S0_NS_GREEN;

        endcase
    end

    always @(posedge clk or posedge reset) begin
        if (reset)
            current_state <= S0_NS_GREEN;
        else
            current_state <= next_state;
    end


    always @(posedge clk or posedge reset) begin
        if (reset) begin
            timer <= 32'd0;
        end
        else if (current_state != next_state) begin
            timer <= 32'd0;
        end
        else if ((current_state == S0_NS_GREEN) &&
                 (AMB_NS || ns_request)) begin
            timer <= 32'd0;
        end
        else if ((current_state == S3_EW_GREEN) &&
                 (AMB_EW || ew_request)) begin
            timer <= 32'd0;
        end
        else begin
            timer <= timer + 1'b1;
        end
    end

    always @(*) begin

        // Default outputs
        NS_RED    = 1'b0;
        NS_YELLOW = 1'b0;
        NS_GREEN  = 1'b0;

        EW_RED    = 1'b0;
        EW_YELLOW = 1'b0;
        EW_GREEN  = 1'b0;

        case (current_state)

            S0_NS_GREEN: begin
                NS_GREEN = 1'b1;
                EW_RED   = 1'b1;
            end

            S1_NS_YELLOW: begin
                NS_YELLOW = 1'b1;
                EW_RED    = 1'b1;
            end

              S2_ALL_RED_1: begin
                NS_RED = 1'b1;
                EW_RED = 1'b1;
            end


            S3_EW_GREEN: begin
                NS_RED   = 1'b1;
                EW_GREEN = 1'b1;
            end

            S4_EW_YELLOW: begin
                NS_RED    = 1'b1;
                EW_YELLOW = 1'b1;
            end

            S5_ALL_RED_2: begin
                NS_RED = 1'b1;
                EW_RED = 1'b1;
            end

            default: begin
                NS_RED = 1'b1;
                EW_RED = 1'b1;
            end

        endcase
    end

endmodule

```

## 🧪 Testbench

Save as `tb_priority_traffic_light_controller.v`.

```verilog
`timescale 1ns / 1ps

module tb_priority_traffic_light_controller;

    reg clk;
    reg reset;
    reg AMB_NS;
    reg AMB_EW;

    wire NS_RED, NS_YELLOW, NS_GREEN;
    wire EW_RED, EW_YELLOW, EW_GREEN;

    priority_traffic_light_controller #(
        .GREEN_TIME(10),
        .YELLOW_TIME(5),
        .RED_TIME(3)
    ) DUT (
        .clk(clk),
        .reset(reset),
        .AMB_NS(AMB_NS),
        .AMB_EW(AMB_EW),
        .NS_RED(NS_RED),
        .NS_YELLOW(NS_YELLOW),
        .NS_GREEN(NS_GREEN),
        .EW_RED(EW_RED),
        .EW_YELLOW(EW_YELLOW),
        .EW_GREEN(EW_GREEN)
    );

    always #5 clk = ~clk;

    initial begin
        $monitor(
            "t=%0t state=%b | AMB_NS=%b AMB_EW=%b | NS(RYG)=%b%b%b | EW(RYG)=%b%b%b",
            $time, DUT.current_state, AMB_NS, AMB_EW,
            NS_RED, NS_YELLOW, NS_GREEN,
            EW_RED, EW_YELLOW, EW_GREEN
        );
    end

    initial begin
        clk = 0;
        reset = 1;
        AMB_NS = 0;
        AMB_EW = 0;

        #20;
        reset = 0;

        $display("TEST 1: Normal traffic cycle");
        #180;

        $display("TEST 2: NS ambulance request");
        AMB_NS = 1;
        #20;
        AMB_NS = 0;
        #100;

        $display("TEST 3: EW ambulance request");
        AMB_EW = 1;
        #20;
        AMB_EW = 0;
        #150;

        $display("TEST 4: Both ambulance requests");
        AMB_NS = 1;
        AMB_EW = 1;
        #20;
        AMB_NS = 0;
        AMB_EW = 0;
        #150;

        $display("TEST 5: Reset");
        reset = 1;
        #20;
        reset = 0;

        #100;
        $display("Simulation complete");
        $finish;
    end

endmodule
```

## ⚙️ Running the Project in Vivado

1. Create a Vivado project targeting the correct Boolean Spartan-7 device.
2. Add `priority_traffic_light_controller.v` under **Design Sources**.
3. Add `tb_priority_traffic_light_controller.v` under **Simulation Sources**.
4. Set `priority_traffic_light_controller` as the synthesis top.
5. Set `tb_priority_traffic_light_controller` as the simulation top.
6. Add the Boolean board `.xdc` constraints file.
7. Run **Run Behavioral Simulation** and inspect the state transitions and outputs.
8. Run synthesis and implementation before generating the bitstream.

The testbench uses shortened timer parameters so that state transitions are visible quickly during simulation. The hardware timing parameters are intended for a 100 MHz clock and should be adjusted as required.


## 🚀 Future Improvements

- Four-direction ambulance detection
- Traffic-density-based adaptive timing
- Pedestrian crossing support
- Seven-segment display for countdown timing
- Hardware demonstration with external traffic LEDs

## 👩‍💻 Technologies Used

- Verilog HDL
- Moore Finite State Machine
- Xilinx Vivado
- Real Digital Boolean Spartan-7 FPGA
- FPGA synthesis and behavioral simulation

---

## ⚠️ Disclaimer

This project was developed **strictly for educational and learning purposes** as part of exploring Verilog HDL, Finite State Machines (FSMs), and FPGA-based digital system design.

The traffic light controller is a learning prototype intended to demonstrate basic traffic sequencing and ambulance-priority logic. It is **not designed, tested, or certified for deployment in real-world traffic control systems**.

The design may require additional safety mechanisms, validation, and regulatory compliance before any real-world application.

## 🎓 Learning Outcomes

Through this project, I explored:
- Designing a Moore FSM using Verilog HDL.
- Implementing timed state transitions.
- Understanding ambulance-priority logic.
- Simulating and debugging digital designs using Xilinx Vivado.
- Mapping RTL signals to physical FPGA pins and LEDs.

**Built with curiosity, coded for learning, and driven by a passion for digital design and FPGA development. 🚦💻**

---

*This project is intended solely for academic demonstration and personal skill development.*

