<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>I2C Controller Implementation - Tang Nano 4K & SSD1306</title>
    <style>
        :root {
            --bg-color: #0d1117;
            --text-color: #c9d1d9;
            --text-muted: #8b949e;
            --border-color: #30363d;
            --accent-color: #58a6ff;
            --code-bg: #161b22;
            --sidebar-bg: #161b22;
            --table-header-bg: #1f242c;
        }

        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Helvetica, Arial, sans-serif, "Apple Color Emoji", "Segoe UI Emoji";
            background-color: var(--bg-color);
            color: var(--text-color);
            line-height: 1.6;
            margin: 0;
            padding: 0;
            display: flex;
        }

        /* Layout Structure */
        #sidebar {
            width: 280px;
            background-color: var(--sidebar-bg);
            border-right: 1px solid var(--border-color);
            position: fixed;
            top: 0;
            bottom: 0;
            left: 0;
            overflow-y: auto;
            padding: 2rem 1.5rem;
            box-sizing: border-box;
        }

        #content {
            margin-left: 280px;
            max-width: 850px;
            padding: 2rem 3rem;
            box-sizing: border-box;
        }

        /* Navigation Links */
        #sidebar h3 {
            font-size: 0.9rem;
            text-transform: uppercase;
            letter-spacing: 0.05em;
            color: var(--text-muted);
            margin-bottom: 1rem;
        }

        #sidebar ul {
            list-style: none;
            padding: 0;
            margin: 0;
        }

        #sidebar li {
            margin-bottom: 0.75rem;
        }

        #sidebar a {
            color: var(--text-color);
            text-decoration: none;
            font-size: 0.95rem;
            transition: color 0.2s ease;
        }

        #sidebar a:hover {
            color: var(--accent-color);
        }

        /* Typography & Elements */
        h1, h2, h3, h4 {
            color: #f0f6fc;
            font-weight: 600;
            margin-top: 1.5rem;
            margin-bottom: 1rem;
        }

        h1 {
            font-size: 2rem;
            border-bottom: 1px solid var(--border-color);
            padding-bottom: 0.5rem;
        }

        h2 {
            font-size: 1.5rem;
            border-bottom: 1px solid var(--border-color);
            padding-bottom: 0.3rem;
            margin-top: 2.5rem;
        }

        h3 { font-size: 1.25rem; }

        p { margin-bottom: 1.2rem; }

        a {
            color: var(--accent-color);
            text-decoration: none;
        }

        a:hover { text-decoration: underline; }

        hr {
            border: 0;
            height: 1px;
            background: var(--border-color);
            margin: 2.5rem 0;
        }

        /* Code Blocks & Pre */
        pre {
            background-color: var(--code-bg);
            border: 1px solid var(--border-color);
            border-radius: 6px;
            padding: 1rem;
            overflow-x: auto;
            font-family: ui-monospace, SFMono-Regular, SF Mono, Menlo, Consolas, Liberation Mono, monospace;
            font-size: 0.875rem;
            color: #e6edf3;
        }

        code {
            background-color: var(--code-bg);
            padding: 0.2rem 0.4rem;
            border-radius: 6px;
            font-family: ui-monospace, SFMono-Regular, SF Mono, Menlo, Consolas, Liberation Mono, monospace;
            font-size: 0.875rem;
        }

        pre code {
            padding: 0;
            background-color: transparent;
            border-radius: 0;
        }

        /* Blockquotes */
        blockquote {
            margin: 0 0 1.2rem 0;
            padding: 0 1rem;
            color: var(--text-muted);
            border-left: 0.25rem solid var(--border-color);
        }

        /* Tables */
        table {
            width: 100%;
            border-collapse: collapse;
            margin-bottom: 1.5rem;
        }

        th, td {
            padding: 0.5rem 1rem;
            border: 1px solid var(--border-color);
            text-align: left;
        }

        th {
            background-color: var(--table-header-bg);
            color: #f0f6fc;
        }

        tr:nth-child(even) {
            background-color: rgba(110, 118, 129, 0.08);
        }

        /* Images and Fallbacks */
        .img-container {
            background-color: var(--code-bg);
            border: 1px solid var(--border-color);
            border-radius: 6px;
            padding: 1rem;
            text-align: center;
            margin: 1.5rem 0;
        }

        .img-container img {
            max-width: 100%;
            height: auto;
            display: block;
            margin: 0 auto 0.5rem auto;
        }

        .img-caption {
            font-size: 0.85rem;
            color: var(--text-muted);
        }

        /* Responsive Design */
        @media (max-width: 900px) {
            body { flex-direction: column; }
            #sidebar {
                position: relative;
                width: 100%;
                height: auto;
                border-right: none;
                border-bottom: 1px solid var(--border-color);
                padding: 1.5rem;
            }
            #content {
                margin-left: 0;
                padding: 1.5rem;
            }
        }
    </style>
</head>
<body>

    <nav id="sidebar">
        <h3>Documentation Hierarchy</h3>
        <ul>
            <li><a href="#overview">1. Project Overview</a></li>
            <li><a href="#i2c-deep-dive">2. Deep Dive: I2C Protocol</a></li>
            <li><a href="#hardware-details">3. Hardware Component Details</a></li>
            <li><a href="#system-architecture">4. System Architecture</a></li>
            <li><a href="#initialization-flow">5. SSD1306 Initialization</a></li>
            <li><a href="#repository-structure">6. Repository Structure</a></li>
            <li><a href="#build-instructions">7. Build and Flash</a></li>
        </ul>
    </nav>

    <main id="content">
        <header>
            <h1>I2C Controller Implementation for SSD1306 OLED Display on Tang Nano 4K FPGA</h1>
            <p>A highly optimized, hardware-level I2C (Inter-Integrated Circuit) master controller written in Verilog/VHDL to interface a <strong>Sipeed Tang Nano 4K FPGA</strong> with a <strong>128x64 SSD1306 OLED Display Driver</strong>.</p>
        </header>

        <hr>

        <section id="overview">
            <h2>🚀 1. Project Overview</h2>
            <p>The objective of this project is to implement a robust, bare-metal I2C master controller directly within the programmable logic fabric of the Tang Nano 4K FPGA. Unlike microcontroller-based implementations that rely on software polling or hardware peripherals (bit-banging / HAL libraries), this design utilizes a dedicated hardware <strong>Finite State Machine (FSM)</strong>. This ensures precise, glitch-free timing controls and maximizes data throughput directly to an SSD1306 OLED screen for real-time telemetry or graphic rendering applications.</p>
        </section>

        <section id="i2c-deep-dive">
            <h2>🔬 2. Deep Dive: The I2C (IIC) Protocol</h2>
            <p>The <strong>Inter-Integrated Circuit (I2C)</strong> protocol is a synchronous, multi-master, multi-slave, packet-switched, single-ended, serial communication bus. It is widely utilized for short-distance, intra-board communication between processors and low-speed peripheral ICs.</p>

            <h3>Physical Layer & Electrical Topography</h3>
            <p>I2C relies strictly on two bidirectional open-drain lines:</p>
            <ul>
                <li><strong>SDA (Serial Data Line):</strong> The line used to transfer data bits between devices.</li>
                <li><strong>SCL (Serial Clock Line):</strong> The line carrying the clock synchronizing the data transfer.</li>
            </ul>
            <p>Because the output drivers are open-drain (or open-collector), they can only pull the lines down to a logic low (0). They cannot drive the lines high (1). External <strong>pull-up resistors</strong> (R<sub>P</sub>) are mandatory to return the lines to a logic high state when no device is pulling them low. This configuration prevents electrical contention or short circuits when multiple devices attempt to drive the bus simultaneously.</p>

            <pre>
                  VCC (e.g., 3.3V)
                   |       |
                  [Rp]    [Rp]  (Pull-up Resistors: 2.2k - 10k Ohms)
                   |       |
     --------------+-------+--------------------- SCL Bus Line
     --------------+-------+--------------------- SDA Bus Line
                   |       |
            +------+-------+------+       +------+-------+------+
            |     SCL     SDA     |       |     SCL     SDA     |
            |                     |       |                     |
            |     I2C MASTER      |       |      I2C SLAVE      |
            |  (Tang Nano 4K)     |       |     (SSD1306)       |
            +---------------------+       +---------------------+
            </pre>

            <h3>Protocol Signaling & Frame Structure</h3>
            <p>An I2C transaction consists of specific operational sequences:</p>
            <ol>
                <li><strong>Idle State:</strong> Both SCL and SDA lines remain tied to logic high (1).</li>
                <li><strong>START Condition (S):</strong> A transition of the SDA line from high to low while SCL is held high. This signals all slave devices on the bus to listen.</li>
                <li><strong>Address Frame (7 bits):</strong> The master sends a unique 7-bit identifier belonging to the targeted slave device.</li>
                <li><strong>Read/Write Bit (R/W):</strong> A single bit appended to the address. 0 denotes a Write operation; 1 denotes a Read operation. (For the SSD1306 write operation, this bit is always 0).</li>
                <li><strong>ACK/NACK Bit (A):</strong> Following every byte, the receiving device must pull SDA low during the 9th clock cycle to issue an Acknowledge (ACK). Leaving SDA high indicates a Not-Acknowledge (NACK).</li>
                <li><strong>Data Frame (8 bits):</strong> Bytes containing data or control payloads are transferred sequentially, MSB (Most Significant Bit) first.</li>
                <li><strong>STOP Condition (P):</strong> A transition of the SDA line from low to high while SCL is held high. This releases the bus.</li>
            </ol>

            <div class="img-container">
                <img src="images/i2c_frame_topology.png" alt="I2C Frame Topology Diagram">
                <div class="img-caption">Figure 1: UI Framework Topology / Link placeholder if using local assets</div>
            </div>

            <pre>
SCL  __   _   _   _   _   _   _   _   _   _       _   _   _   _   _   _   _   _   _   __
       \_/ \_/ \_/ \_/ \_/ \_/ \_/ \_/ \_/ \_____/ \_/ \_/ \_/ \_/ \_/ \_/ \_/ \_/ \_/
SDA  ____                                   _   _________________________________   __
         \_______A6__A5__A4__A3__A2__A1__A0/ \_/ \_D7__D6__D5__D4__D3__D2__D1__D0/ \_/
     | S |         7-Bit Slave Address     |R/W|ACK|          8-Bit Data Byte      |ACK| P |
            </pre>

            <h3>Timing Requirements & Waveforms</h3>
            <p>Data validity on the SDA line is governed by a strict rule: <strong>SDA must remain stable whenever SCL is High.</strong> Changes to the state of the SDA line are only permitted when SCL is Low. The only exceptions to this rule are the START and STOP conditions.</p>

            <div class="img-container">
                <img src="images/i2c_timing_waveform.png" alt="I2C Timing Waveform Diagram">
                <div class="img-caption">Figure 2: Physical Layer Clock Synchronization Tracking</div>
            </div>

            <pre>
SCL         ___________               ___________
                       \             /           \
SDA   ==================X===========X=============X========
      [   Data Allowed  ]           [ Data Allowed ]
      [     to Change   ]           [   to Change  ]
                        |___________|
                         SCL is LOW
            </pre>
        </section>

        <section id="hardware-details">
            <h2>🛠 3. Hardware Component Details</h2>
            
            <h3>Sipeed Tang Nano 4K FPGA</h3>
            <p>The <strong>Sipeed Tang Nano 4K</strong> is a compact development board equipped with the GOWIN Semiconductor <strong>GW1NSR-LV4C</strong> system-on-chip FPGA architecture.</p>
            <ul>
                <li><strong>Logic Elements (LUT4):</strong> 4,608</li>
                <li><strong>Flip-Flops (FF):</strong> 3,456</li>
                <li><strong>Block SRAM (BSRAM):</strong> 180 Kbits</li>
                <li><strong>Clock Unit:</strong> Onboard 27 MHz active crystal oscillator.</li>
                <li><strong>Operating Voltage:</strong> 3.3V I/O bank constraints, matching standard peripheral logic levels.</li>
            </ul>

            <h3>SSD1306 OLED Display Controller</h3>
            <p>The <strong>SSD1306</strong> is a single-chip CMOS OLED/PLED driver with a built-in controller for dot-matrix graphic display systems containing 128 segments and 64 commons.</p>
            <ul>
                <li><strong>Resolution:</strong> 128 x 64 pixels, monochrome emissive displays.</li>
                <li><strong>I2C Device Address:</strong> Commonly factory-strapped to <code>0x3C</code>.</li>
                <li><strong>Control Byte Syntax:</strong> Every data streaming cycle requires an interleaved control byte immediately preceding the target execution command or graphic byte:
                    <ul>
                        <li><strong>Co Bit (Continuation):</strong> If set to 1, the following data payload contains an interleaved stream of command sequences. If set to 0, only data bytes follow.</li>
                        <li><strong>D/C# Bit (Data/Command Selection):</strong> Set to 0 specifies that the next incoming byte is an internal instruction command. Set to 1 specifies that the byte should be pushed into the Graphic Display Data RAM (GDDRAM).</li>
                    </ul>
                </li>
            </ul>

            <div class="img-container">
                <img src="images/ssd1306_gddram_structure.png" alt="SSD1306 GDDRAM Structure Diagram">
                <div class="img-caption">Figure 3: Graphic Display Data RAM Structuring Layout</div>
            </div>
        </section>

        <section id="system-architecture">
            <h2>📐 4. System Architecture & Communication Flow</h2>
            
            <h3>Hardware Interfacing & Wiring</h3>
            <table>
                <thead>
                    <tr>
                        <th>FPGA Pin (Tang Nano 4K)</th>
                        <th>SSD1306 Pin</th>
                        <th>Description</th>
                    </tr>
                </thead>
                <tbody>
                    <tr>
                        <td><strong>VCC (3.3V)</strong></td>
                        <td>VCC</td>
                        <td>Power Input (3.3V Logic level)</td>
                    </tr>
                    <tr>
                        <td><strong>GND</strong></td>
                        <td>GND</td>
                        <td>Shared System Ground</td>
                    </tr>
                    <tr>
                        <td><strong>Pin 31 (Example)</strong></td>
                        <td>SCL</td>
                        <td>Serial Clock I2C Line</td>
                    </tr>
                    <tr>
                        <td><strong>Pin 32 (Example)</strong></td>
                        <td>SDA</td>
                        <td>Open-Drain Serial Data Line</td>
                    </tr>
                </tbody>
            </table>

            <h3>FPGA Finite State Machine (FSM) Design</h3>
            <div class="img-container">
                <img src="images/i2c_fsm_diagram.png" alt="I2C Controller State Machine Diagram">
                <div class="img-caption">Figure 4: FSM Logical State Flow Tracking Schematic</div>
            </div>

            <pre>
        +--------+
------> |  IDLE  | <------------------------------------+
        +--------+                                      |
            | Start Trigger                             |
            v                                           |
        +--------+                                      |
        | START  |                                      |
        +--------+                                      |
            |                                           |
            v                                           |
    +---------------+                                   |
+-> | SEND_BYTE     |                                   |
|   +---------------+                                   |
|       | 8 bits shifted out                            |
|       v                                               |
|   +---------------+                                   |
|   | WAIT_ACK      |                                   |
|   +---------------+                                   |
|       |                                               |
|       |-- [More Bytes To Transfer] -------------------+
|       |                                               |
|       +-- [No More Data] --> +--------+               |
+----------------------------- |  STOP  | --------------+
                               +--------+
            </pre>
        </section>

        <section id="initialization-flow">
            <h2>🛠 5. SSD1306 Initialization & Configuration Flow</h2>
            <p>Before displaying graphical elements on the screen, the FPGA must transmit a precise sequence of configuration commands to the SSD1306 controller during the boot-up phase.</p>

            <pre>
+-------------------------------------------------------+
|                 SYSTEM POWER-ON RESET                 |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  DISPLAY OFF Command [0xAE]                           |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  SET MULTIPLEX RATIO [0xA8, 0x3F]                     |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  SET DISPLAY OFFSET [0xD3, 0x00]                      |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  SET COM PINS HARDWARE CONFIG [0xDA, 0x12]            |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  ENABLE CHARGE PUMP REGULATOR [0x8D, 0x14]            |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  SET ADDRESSING MODE: HORIZONTAL [0x20, 0x00]         |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  DISPLAY ON Command [0xAF]                            |
+-------------------------------------------------------+
                           |
                           v
|             OLED READY FOR DATA OVER I2C              |
+-------------------------------------------------------+
            </pre>
        </section>

        <section id="repository-structure">
            <h2>📂 6. Repository Structure</h2>
            <pre>
├── README.md                  # Comprehensive technical documentation
├── index.html                 # Web documentation module (This File)
├── doc/                        # Architectural spec sheets & graphics references
│   └── images/                 # Stored structural schematics and waveform graphics
├── src/                        # FPGA Hardware Source Code
│   ├── rtl/                    # Register-Transfer Level Core Modules
│   │   ├── i2c_master.v        # Low-level I2C physical layer state machine core
│   │   └── top.v               # Top-level hardware module connecting clock & physical pins
│   └── constraints/            # Synthesis Physical Allocation Definitions
│       └── pinout.cst          # Physical constraint assignments mapping to Tang Nano 4K
└── sim/                        # Testbench validation files
    └── i2c_master_tb.v         # Functional simulation verification testbench
            </pre>
        </section>

        <section id="build-instructions">
            <h2>🛠 7. How to Build, Synthesize, and Flash</h2>
            <p>To compile this design and load it onto your Sipeed Tang Nano 4K FPGA:</p>
            <ol>
                <li><strong>Initialize Gowin Project Setup:</strong> Open your Gowin EDA interface and target the specific <code>GW1NSR-LV4C</code> device model (Package: <code>QN48P</code>).</li>
                <li><strong>Incorporate Design Constraints:</strong> Ensure that the physical pin assignments match inside your <code>pinout.cst</code> constraint layout file:
                    <pre>
IO_LOC "clk" 35;       // System 27 MHz active crystal clock pin
IO_LOC "i2c_scl" 31;   // SCL mapping pin output 
IO_LOC "i2c_sda" 32;   // SDA mapping pin output
IO_PORT "i2c_sda" IO_TYPE=LVCMOS33 PULL_MODE=UP DRIVE=8;
IO_PORT "i2c_scl" IO_TYPE=LVCMOS33 PULL_MODE=UP DRIVE=8;
                    </pre>
                </li>
                <li><strong>Run Synthesis and Route:</strong> Execute the toolchain compiler steps to generate your target hardware structural layer binary configuration <code>.fs</code> file payload.</li>
                <li><strong>Flash the Fabric:</strong> Connect your device over an FTDI programmer or standard USB-C cable layer and write to the non-volatile embedded flash storage bank using the Gowin Programmer terminal tool.</li>
            </ol>
        </section>
    </main>

</body>
</html>
