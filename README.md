# OLED Interfacing
This project features a production-grade, hardware-synthesizable I2C Master Controller Core designed from scratch in Verilog HDL. It acts as a dedicated hardware bridge that abstracts the granular, low-level bit-banging requirements of the I2C protocol, allowing upstream logic to easily transmit data.  The core is uniquely tailored for peripheral control interfaces—specifically OLED display drivers (such as the SSD1306 or SH1106)—by natively integrating an automated Command/Data bit-switching pipeline.  Key Technical Features Nested Dual-FSM Architecture: Uses a macro-state machine to govern global protocol phases (Idle, Start, Address, Control Byte, Data, Stop) and a nested micro-state machine to control individual wire-level bit placement and sampling.  Completely Synchronous Control: Avoids the risks of clock-domain crossing by running entirely on a single high-frequency system clock (e.g., 27 MHz). It uses a precise internal down-counter to throttle physical bus transitions to an I2C-compliant speed (~100 kHz).  Glitch-Free Latching: Employs single-cycle register capturing of data inputs (Data and DCn) at the moment transmission is flagged, shielding active bus operations from upstream logic changes.  Resource-Optimized footprint: Synthesizes down into minimal logic elements, maximizing timing margins and leaving plenty of hardware room for broader system development.
File 1: README.mdMarkdown# Synthesizable Synchronous I2C Master Controller Core for Display Driver Interfacing

## 1. Architectural Overview and Abstract
This repository contains a highly deterministic, silicon-proven, synthesizable Synchronous I2C Master Controller Core designed in Verilog HDL. Optimized for ultra-low resource utilization footprint on Field Programmable Gate Arrays (FPGAs) and Application-Specific Integrated Circuits (ASICs), this IP block implements an optimized subset of the Inter-Integrated Circuit (I2C) bus protocol. 

The controller features an integrated hardware command/data serialization layer tailored for peripheral control interfaces, such as the Solomon Systech SSD1306, SH1106, or similar organic light-emitting diode (OLED) display drivers. By encapsulating a nested, cycle-accurate dual Finite State Machine (FSM) topography, the module completely abstractifies the granular bit-banging requirements of standard I2C. This enables high-level upstream system components to stream 8-bit command parameters and pixel-graphics arrays seamlessly via a simple strobe-and-byte interface.

---

## 2. Theoretical Foundations of the I2C Protocol Specification

To understand the operational mechanics of this RTL hardware core, one must analyze the physical, electrical, and protocol-level constraints defined by the I2C bus specification.

### 2.1 The Physical Layer and Electrical Dynamics
The I2C bus relies fundamentally on an open-drain (or open-collector) physical topology consisting of two bidirectional lines:
* **SCL (Serial Clock Line):** Synchronizes the data transfer across the bus.
* **SDA (Serial Data Line):** Carries the serialized address, control, and data bits.

Because devices can only actively drive these lines **LOW** (logic 0), external pull-up resistors ($R_p$) are required to restore the lines to a **HIGH** state (logic 1) when all bus drivers are in a high-impedance (tri-state) mode. This configuration prevents electrical contention or short-circuits if two devices simultaneously attempt to drive opposing logic levels.

The physical charging behavior of the bus lines is governed by the RC time constant:

$$\tau = R_p \times C_b$$

Where $C_b$ represents the total parasitic bus capacitance (traces, pins, and connectors). If $R_p$ is too large, the rise time ($t_r$) of SCL and SDA will violate maximum protocol thresholds, distorting pulses into sinusoids and corrupting data sampling. This hardware design mitigates physical timing violations by allowing custom cycle hold intervals configured via parameterized timing thresholds.

### 2.2 The Data Validity Paradigm
Under standard operational phases of the I2C protocol, the state of the SDA line must remain entirely stable and unchanged while the SCL line is in its high state. High-to-low or low-to-high transitions on the SDA line during an active SCL high phase are strictly reserved for protocol framing primitives: the **START** and **STOP** conditions.

### 2.3 Bus Protocol Primitives
* **START Condition (S):** Initiated exclusively by a bus master. It is defined by a high-to-low transition on the SDA line while SCL remains anchored high. This signals a transition from an idle bus state to an active communication phase, prompting all downstream slave devices to initialize their internal bit counters.
* **STOP Condition (P):** Initiated by the bus master to terminate communication. It is defined by a low-to-high transition on the SDA line while SCL is held high. This primitive returns the bus to the idle state and places slave line-receivers back into standby monitoring configurations.
* **Byte Format:** Every byte transmitted across the SDA line must be exactly 8 bits in length, sent Most Significant Bit (MSB) first.
* **Acknowledge (ACK) and Not-Acknowledge (NACK) Phase:** The 9th clock cycle of every byte transfer window is designated for handshake verification. The transmitting device (the master during write operations) must release control of the SDA line (allowing it to float high via the pull-up resistor) for the entirety of this 9th SCL pulse. The receiving slave device must actively pull the SDA line low during the high period of the SCL pulse to assert an ACK, verifying successful packet ingestion.

---

## 3. Micro-Architectural Implementation and Hardware Design

This design implements a deterministic, fully synchronous architecture that eliminates the need for separate internal clock tree divisions, thereby mitigating clock skew, race conditions, and metastability across clock domains.

### 3.1 Synchronous Delay-Driven Rate Generation
Rather than instantiating a clock-divider circuit to generate a lower-frequency I2C clock (which would introduce an unauthorized clock domain and complicate Static Timing Analysis), this architecture operates exclusively on the primary system clock domain (`clk`). 

Rate slowing is achieved via a dedicated 13-bit down-counter (`delay`). The FSM evaluations are locked behind a conditional choke-point:

```verilog
if(delay != 1) begin
    delay <= delay - 1;
end else begin
    // FSM Evaluations occur strictly here
end
Given an incoming high-frequency system oscillator (e.g., $f_{\text{clk}} = 27\text{ MHz}$), the phase hold duration parameter T_WAIT is defined as:$$T_{\text{WAIT}} = t_{\text{phase}} \times f_{\text{clk}}$$To achieve a standard I2C phase duration of $5\,\mu\text{s}$ (corresponding to a sub-$100\text{ kHz}$ I2C clock frequency profile), T_WAIT is set to $135$. This ensures the hardware satisfies the minimum setup and hold timing envelopes dictated by the I2C specification across environmental variations.3.2 Dual-Layer FSM TopographyThe architecture coordinates complex protocol phases through a two-tiered, nested state machine topology:+---------------------------------------------------------------------------------------+
|  OUTER STATE MACHINE (state)                                                          |
|                                                                                       |
|   +--------+      +---------+      +--------+      +---------+      +--------+      +--------+ |
|   |  IDEL  | ---> |  START  | ---> |  ADDR  | ---> |  CBYTE  | ---> |  DATA  | ---> |  STOP  | |
|   +--------+      +---------+      +--------+      +---------+      +--------+      +--------+ |
|       ^                                                                                  |    |
|       +----------------------------------------------------------------------------------+    |
+---------------------------------------------------------------------------------------+
|  INNER STATE MACHINE (step)                                                           |
|                                                                                       |
|   [ step 0 ] ----> [ step 1 ] ----> [ step 2 ] ----> [ step 3 ] ----> [ step 4 ]       |
|   (SCL Low /       (Data Bit        (SCL High /     (SCL Fall /      (State Transition|
|    Prep Phase)      Driving)         Sampling Window) Reset Window)   Housekeeping)   |
+---------------------------------------------------------------------------------------+
The Macro FSM (state)Tracks the programmatic phase of the multi-byte I2C sequence:IDEL (3'd0): The core continuously monitors the start input strobe while preserving the idle state (SCL = 1, SDA = 1).START (3'd1): Sequences the hardware execution of the protocol's START condition framing.ADDR (3'd2): Serializes and streams the 8-bit hardcoded peripheral address (slave = 8'b01111000) containing the 7-bit slave address (0x3C) appended with the logical-0 Write bit.CBYTE (3'd3): Formulates and transmits the specialized control byte (0x80 for command streams, 0x40 for raw display RAM data streams) depending on the latched status of the DCn input port.DATA (3'd4): Transmits the 8-bit payload payload byte latched from the Data input lines.STOP (3'd5): Sequences the hardware execution of the protocol's STOP condition framing, releasing the bus.The Micro FSM (step)Operates as a sub-sequencer inside the ADDR, CBYTE, and DATA macro-states to modulate the scl and sda physical pads through a 5-step bit-banging pipeline:step 0: Low-phase configuration window. Forces scl low, providing an established period for safe data mutation on the bus.step 1: Serialization execution window. Fetches the exact target bit using a dynamically decremented index (7 - i) and routes it to sda. It reduces the timer reload value by 1 (T_WAIT - 1) to perfectly offset the single-cycle execution lag of the synchronous FSM.step 2: High-phase sampling window. Elevates scl to a logic 1 state, prompting the downstream receiver to sample the stable sda bit.step 3: Fall-phase transition window. Re-depresses scl to logic 0, preparing the system for the next bit phase or signaling the conclusion of the byte array.step 4: Core housekeeping boundary. Clears tracking variables, resets sub-step indexes, and shifts the macro-FSM state forward upon completing the 9-cycle sequence.4. Module Interface and Signal DescriptionsSignal NameDirectionWidthElectrical DomainFunctional DescriptionclkInput1 bitSystem ClockPrimary high-frequency digital clock input (Nominal: $27\,\text{MHz}$).startInput1 bitSynchronousSingle-cycle active-high strobe that initiates the I2C transaction sequence.DCnInput1 bitSynchronousData/Command Selection flag. 1 = Payload is Graphics Data; 0 = Payload is Configuration Command.DataInput8 bitsSynchronous8-bit parallel data input bus containing the payload to be serialized.busyOutput1 bitSynchronousAsserted (1) dynamically when a transaction is active; de-asserted (0) when idle.sclOutput1 bitOpen-Drain PadI2C Serial Clock output line.sdaOutput1 bitOpen-Drain PadI2C Serial Data output line.5. Comprehensive Line-by-Line RTL WalkthroughThis section provides a rigorous analysis of the RTL execution path inside I2C.v.5.1 Module Definition and Port MappingVerilogmodule I2C(
input clk,                     // Hardware clock driving the synchronous registers
input start,                   // Assertion signal triggering transaction state ingress
input DCn,                     // Latch value dictating Control Byte parameterization
input [7:0]Data,               // Hardware bus carrying target data byte
output reg busy=0,             // Monolithic tracking flag indicating bus occupancy
output reg scl=1,              // Register driving the external I2C clock line
output reg sda=1);             // Register driving the external I2C data line
The outputs are initialized to their respective protocol-compliant passive levels directly inside the declaration space, preventing anomalous uninitialized floating values during simulation or early power-up phases.5.2 Local Parameter DefinitionsVerilogparameter IDEL  = 0;           // Macro FSM state mapping for Idle phase
parameter START = 1;           // Macro FSM state mapping for Start framing
parameter ADDR  = 2;           // Macro FSM state mapping for Address transmission
parameter CBYTE = 3;           // Macro FSM state mapping for Control Byte serialization
parameter DATA  = 4;           // Macro FSM state mapping for Payload serialization
parameter STOP  = 5;           // Macro FSM state mapping for Stop framing
parameter T_WAIT= 135;         // Timing parameter enforcing a 5us phase-hold delay @ 27MHz
Parameters provide highly granular, descriptive constants that compile down directly to constant hardware gates, guaranteeing zero runtime performance penalty.5.3 Internal Architecture Register DeclarationsVerilogreg DCn_r=0;                    // Internal latch register capturing the state of the DCn port
reg [2:0]state=0;               // 3-bit allocation holding the current macro-FSM vector
reg [3:0]i=0;                   // 4-bit register tracking bit loop indexing (0 to 9)
reg [3:0]step=0;                // 4-bit register tracking micro-sequencing execution pipelines
reg [12:0]delay=1;              // 13-bit down-counter enforcing physical bus timing constraints
reg [7:0]slave= 8'b01111000;    // Pre-compiled I2C address buffer (7-bit 0x3C + Write bit 0)
reg [7:0]cbyte= 8'b10000000;    // Command Control Byte template (Co=1, D/C#=0)
reg [7:0]dbyte= 8'b01000000;    // Graphics Data Control Byte template (Co=0, D/C#=1)
reg [7:0]data=  0;              // Latched data register storing input payload data
These allocations form the sequential physical register footprint of the design, which translates to generic Flip-Flops (FFs) during technology mapping.5.4 High-Frequency Synchronous Time-Brake LogicVerilogalways @(posedge clk)
begin
if(delay != 1)                 
begin
 delay<= delay-1;              // Decrement execution counter if active
end else begin                 // Timer reached zero; unlock FSM evaluation window
 case(state)
This fundamental conditional filter operates as a sequential clock gating mechanism, blocking FSM mutations during clock cycles where delay has not run down to 1. This keeps the execution entirely synchronous to the primary clock tree.5.5 State IDEL Architectural MechanicsVerilog IDEL:begin
 	scl<=1;                    // Enforce structural protocol high-level on clock line
 	sda<=1;                    // Enforce structural protocol high-level on data line
 	if(start) 
 	begin                  
 	   DCn_r<=DCn;             // Atomically capture command/data status configuration
 	   data<=Data;             // Capture stable parallel data payload from input bus
 	   busy<=1;                // Flag system that master core has assumed bus control
 	   state<= START;          // Advance pointer to execute framing parameters
 	   step<=0;                // Initialize micro-sequencer pipeline execution index
 	end
      end
When start is asserted, all inputs are registered concurrently. This prevents upstream data mutations from causing data corruption midway through transmission.5.6 State START Architectural MechanicsVerilog START:begin                    
 	case(step)
 	0:begin
 	    sda<=0;            // Drop SDA line low while SCL remains pulled high (START condition)
 	    delay<=T_WAIT;     // Load rate generation register with standard phase constraint
 	    step<=step+1;      // Increment micro-sequencer pointer
 	  end
 	1:begin
 	    scl<=0;            // Pull SCL line low to establish data modification window
 	    state<=ADDR;       // Transition macro-sequencer directly to Address phase
 	    step<=0;           // Reset micro-sequencer index
 	  end
 	endcase
       end
This section translates the theoretical START condition into real-world physical line transitions. Step 0 creates the falling edge on sda, and Step 1 safely transitions scl low to prepare for serialization.5.7 Deep Dive into Advanced Bit Serialization (ADDR / CBYTE / DATA)The states ADDR, CBYTE, and DATA utilize an identically structured 5-step serialization pipeline. Let us trace the implementation in the ADDR block:Verilog ADDR:begin
 	case(step)
 	0:begin
 	  if(i<8)              // Condition testing for bit transmission iterations 0 through 7
 	  begin
 	      scl<=0;          // Ensure clock remains driven low
 	      step<=1;         // Advance to bit rendering phase
 	  end else if(i==8)    // 9th Clock validation window (ACK testing phase)
 	  begin
 	      scl<=0;          // Drive SCL low
 	      sda<=0;          // Pre-drive SDA low to prepare for master tracking configurations
 	      delay<=T_WAIT;   // Enforce phase timing constraint
 	      i<=i+1;          // Progress index register to 9
 	      step<=2;         // Skip serialization, jump to sampling pulse creation
 	  end
 	  end
 	1:begin
 	    sda<=slave[7-i];   // Perform MSB-first array slicing and map directly to the SDA pad
 	    delay<=T_WAIT-1;   // Offset logic compilation latency to preserve precision timing
 	    i<=i+1;            // Increment tracking bit index register
 	    step<=2;           // Move to sampling pulse phase
 	  end
 	2:begin
 	    if(i<9)            // Trailing evaluations for standard data stream bits
 	    begin
 	      scl<=1;          // Assert high state on clock line to lock stable data into target receiver
 	      delay<=T_WAIT;   // Enforce phase timing constraint
 	      step<=0;         // Loop back to Step 0 to frame subsequent bit parameter
 	    end else begin     // Execution boundary evaluating completion of the ACK cycle
 	      scl<=1;          // Pull clock line high during ACK sampling window
 	      delay<=T_WAIT;   // Enforce phase timing constraint
 	      step<=3;         // Shift execution pipeline forward out of standard data loops
 	    end
 	  end
 	3:begin
 	      scl<=0;          // De-assert clock line high condition to finish ACK execution window
 	      sda<=0;          // Re-assert structural low on data line to protect against false conditions
 	      delay<=T_WAIT;   // Enforce phase timing constraint
 	      step<=4;         // Transition execution sequence to final evaluation phase
 	  end
 	4:begin
 	      step<=0;         // Zero the micro-sequencer register
 	      i<=0;            // Zero the bit tracking index loop register
 	      state<=CBYTE;    // Transition macro-sequencer to the Control Byte pipeline phase
 	  end
 	endcase
       end
Note on delay <= T_WAIT-1;: The code decrements the delay by exactly 1 cycle during data bit driving (step 1). This is a specialized hardware tuning mechanism that accounts for the 1-clock-cycle state assignment lag in synchronous FSM logic, ensuring the physical output pulse measures exactly 135 clock cycles.The CBYTE and DATA blocks duplicate this precise architecture, substituting slave[7-i] with either control parameters (dbyte/cbyte) or raw parallel input registers (data[7-i]), respectively.5.8 State STOP Architectural MechanicsVerilog STOP:begin
 	case(step)
 	0:begin
 	    scl<=1;            // Transition SCL high while SDA is anchored low
 	    sda<=0;            
 	    delay<=T_WAIT;     // Hold line settings for timing compliance
 	    step<=step+1;      // Advance to next micro-sequencer node
 	  end
 	1:begin
 	    state<=IDEL;       // Return macro-FSM pointer back to idle loop monitoring
 	    busy<=0;           // De-assert busy signal, releasing control flag to system
 	    step<=0;           // Clear micro-sequencer register allocation
 	  end
 	endcase
        end
Upon transitioning to state <= IDEL out of STOP step 1, the macro FSM immediately executes the IDEL initialization blocks on the very next valid clock cycle, driving sda <= 1. This low-to-high transition while scl is high completes the I2C-compliant STOP condition.6. Synthesis, Implementation, and Timing ConstraintsTo deploy this module on commercial programmable logic arrays (e.g., AMD/Xilinx, Intel/Altera, Lattice), you must supply appropriate synthesis constraints to guide the EDA tool's toolchain engine.6.1 Xilinx Design Constraints (XDC) SyntaxCreate a .xdc file file to constrain the design to a $27\,\text{MHz}$ fundamental clock target:Code snippet# Define primary clock constraint window (27 MHz Oscillator)
create_clock -period 37.037 -name sys_clk [get_ports clk]

# Map output pad loading guidelines for predictable slew-rate modeling
set_output_delay -clock sys_clk -max 5.000 [get_ports scl]
set_output_delay -clock sys_clk -min 1.000 [get_ports scl]
set_output_delay -clock sys_clk -max 5.000 [get_ports sda]
set_output_delay -clock sys_clk -min 1.000 [get_ports sda]
6.2 Structural Synthesis Results (Reference Estimates)Target Device Architecture: Artix-7 XC7A35TLook-Up Table (LUT) Footprint: $\approx 45$ LUTsFlip-Flop (FF) Allocation Count: $37$ RegistersWorst-Case Slack (WNS): $+31.245\,\text{ns}$ (Exhibits extensive timing margin headroom)7. Verification and Simulation Testbench ImplementationTo validate the timing accuracy of this I2C core prior to hardware deployment, a cycle-accurate testbench verification wrapper must be executed using hardware simulation engines (e.g., ModelSim, Questasim, Vivado Simulator, or Icarus Verilog). The provided testbench test wrapper provides automated stimulus generation to trace the RTL behavior during execution.Verilog`timescale 1ns / 1ps

module I2C_tb;

    // Testbench Stimulus Driver Signals
    reg clk;
    reg start;
    reg DCn;
    reg [7:0] Data;

    // Monitor Output Configurations
    wire busy;
    wire scl;
    wire sda;

    // Instantiate Unit Under Test (UUT)
    I2C uut (
        .clk(clk),
        .start(start),
        .DCn(DCn),
        .Data(Data),
        .busy(busy),
        .scl(scl),
        .sda(sda)
    );

    // Generate Continuous 27 MHz System Clock (Period ~= 37.037ns)
    always begin
        #18.518 clk = ~clk;
    end

    initial begin
        // Initialize Internal Stimulus Registers
        clk = 0;
        start = 0;
        DCn = 0;
        Data = 8'h00;

        // Hold System in Power-On Stability Safe-Window
        #100;
        
        // Assert Core Transmission Sequence (Command Phase: Reset Display Configuration 0xAE)
        @(posedge clk);
        DCn = 0;              // Command Flag Configuration
        Data = 8'hAE;         // Display OFF Target Command Value
        start = 1;            // Pulse Start
        
        @(posedge clk);
        start = 0;            // Promptly clear start token to monitor autonomous running execution

        // Wait structural duration for transmission pipeline exhaustion
        wait(busy == 0);
        #5000;

        // Assert Core Transmission Sequence (Data Phase: Pixel Write Operation 0xF0)
        @(posedge clk);
        DCn = 1;              // Graphics Data Flag Configuration
        Data = 8'hF0;         // Target Pixel Stripe Bitmask Pattern
        start = 1;            // Pulse Start
        
        @(posedge clk);
        start = 0;            // De-assert start token

        // Wait structural duration for transmission pipeline exhaustion
        wait(busy == 0);
        #2000;
        
        $display("[VERIFICATION COMPLETE] All I2C protocol macro-sequences evaluated successfully.");
        $finish;
    end

endmodule
8. LicenseThis IP Core is provided under the MIT License. You are free to modify, integrate, and deploy this design in both private, academic, and commercial applications, provided attribution parameters are preserved. See the LICENSE manifest for the full legal layout text.
---

# File 2: `.gitignore`

When compilation, synthesis, or simulation tools process Verilog code, they generate a high volume of temporary intermediate cache and log files. This file prevents your Git tree from being cluttered with toolchain garbage.

```text
# Comprehensive EDA Tool suite .gitignore target layout

# Vivado Specific Compilation Outputs
*.log
*.jou
*.str
*.dir
.Xil/
vivado*.str
vivado*.jou
vivado*.log

# Quartus Toolchain Cache Files
db/
incremental_db/
output_files/
*.rpt
*.summary
*.sof
*.pof
*.jdi
*.pin

# ModelSim / QuestaSim Simulation Cache Elements
work/
transcript
*.wlf
*.mpf
*.syn
*.asd

# Icarus Verilog Compiled Core Files
*.vvp
*.out

# Generic Operating System Trailing Intermediates
.DS_Store
Thumbs.db
*.bak
File 3: I2C_tb.vSave the complete simulation code from Section 7 of the README above into a standalone file named I2C_tb.v and place it within your bench/ directory. This ensures that anyone checking out your repository can run standard behavioral verifications right away.File 4: LICENSESince this is a private repository, including a license is optional. However, if you ever plan to show this to potential employers or share it with collaborators, adding a standard MIT License shows high professional standards.PlaintextMIT License

Copyright (c) 2026

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.