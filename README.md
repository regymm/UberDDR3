# Specifications and Brief Description
This DDR3 controller was originally designed to be used on the [10-Gigabit Ethernet Project](https://github.com/ZipCPU/eth10g) for an 8-lane x8 DDR3 module running at 800 MHz DDR, but this is now being designed to be a more general memory controller with multiple supported FPGA boards. This is a 4:1 memory controller with configurable timing parameters and mode registers so it can be configured to any DDR3 memory device. The user-interface is the basic Wishbone.

This memory controller is optimized to maintain a high data throughput and continuous sequential burst operations. The controller handles the reset sequence, refresh sequence, mode register configuration, bank status tracking, timing delay tracking, command issuing, and the PHY's internal calibration. The PHY's internal calibration handles the bitslip training, read dqs alignment via MPR (read calibration), write dqs alignment via write leveling (write calibration), and also an optional comprehensive read/write test. The internal read/write test include a burst access, random access, and alternating read-write access tests. Only if no error is found on these tests will the calibration end and user can start accessing the wishbone interface. 

This design is [formally verified](https://github.com/AngeloJacobo/DDR3_Controller/wiki/User-Documentation#lint-and-formal-verification) and [simulated using the Micron DDR3 model](https://github.com/AngeloJacobo/DDR3_Controller/wiki/User-Documentation#simulation).

# Getting Started
The recommended way to instantiate this IP is to use the top module [`rtl/ddr3_top.v`](https://github.com/AngeloJacobo/DDR3_Controller/blob/main/rtl/ddr3_top.v), a template for instantiation is also included in that file. Steps to include this DDR3 memory controller IP is to instantiate design, create the constraint file, then edit the localparams. 

## :heavy_check_mark: Instantiate Design
The first thing to edit are the **top-level parameters**:

| Parameter | Function |
| :---:        |     :---       | 
| CONTROLLER_CLK_PERIOD   | clock period of the controller interface in picoseconds. Tested values range from `12_000` ps (83.33 MHz) to `10_000` ps (100 MHz).     | 
| DDR3_CLK_PERIOD     | clock period of the DDR3 RAM device in picoseconds which must be 1/4 of the `CONTROLLER_CLK_PERIOD`. Tested values range from `3_000` ps (333.33 MHz) to `2_500` ps (400 MHz).     |
| ROW_BITS     | width of row address. Use chapter _2.11 DDR3 SDRAM Addressing_ from [JEDEC DDR3 doc (page 15)](https://www.jedec.org/sites/default/files/docs/JESD79-3F.pdf) as a guide. Possible values range from `12` to `16`.     |
| COL_BITS     | width of column address. Use chapter _2.11 DDR3 SDRAM Addressing_ from [JEDEC DDR3 doc (page 15)](https://www.jedec.org/sites/default/files/docs/JESD79-3F.pdf) as a guide. Possible values range from `10` to `12`.     |
| BA_BITS     | width of bank address. Use chapter _2.11 DDR3 SDRAM Addressing_ from [JEDEC DDR3 doc (page 15)](https://www.jedec.org/sites/default/files/docs/JESD79-3F.pdf) as a guide. Usual value is `3`.      |
| DQ_BITS     | device width. Use chapter _2.11 DDR3 SDRAM Addressing_ from [JEDEC DDR3 doc (page 15)](https://www.jedec.org/sites/default/files/docs/JESD79-3F.pdf) as a guide. Possible values are `4`, `8`, or `16`.  <sup>[[1]](https://github.com/AngeloJacobo/DDR3_Controller/wiki/User-Documentation#note) </sup>   |
| LANES     | number of DDR3 device to be controlled. This depends on the DDR3 module used. <sup>[[1]](https://github.com/AngeloJacobo/DDR3_Controller/wiki/User-Documentation#note) </sup> |
| AUX_WIDTH | width of auxiliary line. Value must be >= 4.  <sup>[[2]](https://github.com/AngeloJacobo/DDR3_Controller/wiki/User-Documentation#note) </sup>  |
| WB2_ADDR_BITS | width of 2nd wishbone address bus for debugging (only relevant if SECOND_WISHBONE = 1).   |
| WB2_DATA_BITS  | width of 2nd wishbone data bus for debugging (only relevant if SECOND_WISHBONE = 1).  |
| OPT_LOWPOWER  | _has no effect yet_ |
| OPT_BUS_ABORT  | _has no effect yet_ |
| MICRON_SIM  | set to 1 if used in Micron DDR3 model to shorten power-on sequence, otherwise 0. |
| ODELAY_SUPPORTED | set to 1 if ODELAYE2 primitive is supported by the FPGA, otherwise 0.  <sup>[[3]](https://github.com/AngeloJacobo/DDR3_Controller/wiki/User-Documentation#note) </sup>  |
| SECOND_WISHBONE | set to 1 if 2nd wishbone for debugging is needed , otherwise 0.|


***

After the parameters, connect the ports of the top module to your design. Below are the **ports for clocks and reset**:
| Ports | Function |
| :---:        |     :---       | 
| i_controller_clk   | clock of the controller interface with period of `CONTROLLER_CLK_PERIOD` | 
| i_ddr3_clk   | clock of the DDR3 interface with period of `DDR3_CLK_PERIOD` | 
| i_ref_clk   | reference clock for IDELAYCTRL primitive with frequency of 200 MHz | 
| i_ddr3_clk_90   | clock required only if ODELAY_SUPPORTED = 0, otherwise can be left unconnected. Has a period of `DDR3_CLK_PERIOD` with 90° phase shift. | 
| i_rst_n   | Active-low synchronous reset for the entire DDR3 controller and PHY | 

It is recommended to generate all these clocks from a single PLL or clock-generator.

***

Next are the **main wishbone ports**:

| Ports | Function |
| :---:        |     :---       | 
| i_wb_cyc   | Indicates if a bus cycle is active. A high value (1) signifies normal operation, while a low value (0) signals the cancellation of all ongoing transactions. | 
| i_wb_stb   |  Strobe or transfer request signal. It's asserted (set to 1) to request a data transfer. | 
| i_wb_we   | Write-enable signal. A high value (1) indicates a write operation, and a low value (0) indicates a read operation. | 
| i_wb_addr  | Address bus. Used to specify the address for the current read or write operation. Formatted as {row, bank, column}. | 
| i_wb_data   | Data bus for write operations. In a 4:1 controller, the data width is 8 times the DDR3 pins `8`x`DQ_BITS`x`LANES`. | 
| i_wb_sel   | Byte select for write operations. Indicates which bytes of the data bus are to be overwritten for the write operation. | 
| o_wb_stall   | Indicates if the controller is busy (1)and cannot accept any new requests. | 
| o_wb_ack   |  Acknowledgement signal. Indicates that a read or write request has been completed. | 
| o_wb_data   | Data bus for read operations. Similar to `i_wb_data`, the data width for a 4:1 controller is 8 times the DDR3 pins `8`x`DQ_BITS`x`LANES`. |

***

Below are the **auxiliary ports** associated with the main wishbone. This is not required for normal operation, but is intended for AXI-interface compatibility *which is not yet available*:  
| Ports | Function |
| :---:        |     :---       | 
| i_aux   | Request ID line with width of `AUX_WIDTH`. The Request ID is retrieved simultaneously with the strobe request. | 
| o_aux   | Request ID line with width of `AUX_WIDTH`. The Request ID is sent back concurrently with the acknowledgement signal.  | 


***

After main wishbone port are the **second-wishbone ports**. This interface is only for debugging-purposes and would normally not be needed thus can be left unconnected by setting `SECOND_WISHBONE` = 0. The ports for the second-wishbone is very much the same as the main wishbone.

Next are the **DDR3 I/O ports**, these will be connected directly to the top-level pins of your design thus port-names must match what is indicated on your constraint file. You do not need to understand what each DDR3 I/O ports does but if you're curious, details on each DDR3 I/O pins are described on _2.10 Pinout Description_ from [JEDEC DDR3 doc (page 13)](https://www.jedec.org/sites/default/files/docs/JESD79-3F.pdf).

Finally are the **debug ports**, these are connected to relevant registers containing information on current state of the controller. Trace each `o_debug_*` inside `ddr3_controller.v` to edit the registers to be monitored.

## :heavy_check_mark: Create Constraint File
* One example of constraint file is from the [Kintex-7 Ethernet Switch Project](https://github.com/AngeloJacobo/DDR3_Controller/blob/main/kintex_switch_files/kluster.xdc#L227-L389)  <sup>[[4]](https://github.com/AngeloJacobo/DDR3_Controller/wiki/User-Documentation#note) </sup>, highlighted are all the DDR3 pins. This constraint file assumes a dual-rank DDR3 RAM (thus 2 pairs of `o_ddr3_clk`, `o_ddr3_cke`, `o_ddr3_s_n`, and `o_ddr3_odt`)  with 8 lanes of x8 DDR3 (thus 8 `o_ddr3_dm`, 8 `io_ddr3_dqs`, and 64 `io_ddr3_dq`). The constraint file also has [set_property](https://github.com/AngeloJacobo/DDR3_Controller/blob/main/kintex_switch_files/kluster.xdc#L453-L457) required for proper operation. The property `INTERNAL_VREF` must be set to half of the bank voltage (1.5V thus set to `0.75`). The property `BITSTREAM.STARTUP.MATCH_CYCLE` ([page 240 of UG628: Command Line Guide](https://docs.xilinx.com/v/u/en-US/devref)) is verified to work properly when value is set to `6`. Kintex-7 has HP bank where the DDR3 is connected thus allow the use of DCI (Digitally-Controlled Impedance) for impedance matching by using `SSTL15_T_DCI` type of `IOSTANDARD`.

* Another example of constraint file is for the [Arty-S7 project](https://github.com/AngeloJacobo/DDR3_Controller/blob/main/testbench/ARTY_S7/Arty-S7-50-Master.xdc#L87-L349), highlighted are the DDR3 pins. The Arty-S7 has x16 DDR3 and it works like two x8 (thus 2 `ddr3_dm`, 2 `ddr3_dqs`, and 16 `io_ddr3_dq`) <sup>[[1]](https://github.com/AngeloJacobo/DDR3_Controller/wiki/User-Documentation#note) </sup>. Arty-S7 only has HR bank where the DDR3 is connected, this restricts the design to use on-chip split-termination [(UG471 7-Series Select Guide page 33)](https://docs.xilinx.com/v/u/en-US/ug471_7Series_SelectIO) for impedance matching instead of DCI used in HP banks. `IN_TERM UNTUNED_SPLIT_50` signifies that the input termination is set to an untuned split termination of 50 ohms. The constraint file was  easily created by retrieving the pin constraints generated by the Vivado Memory Interface Generator (MIG) together with the [`.prj` file](https://github.com/Digilent/vivado-boards/blob/master/new/board_files/arty-s7-50/B.0/mig.prj#L47-L96) provided by Digilent for Arty-S7. The generated `.xdc` file by the MIG can be located at `[vivado_proj].gen/sources_1/ip/mig_7series_0/mig_7series_0/user_design/constraints/mig_7series_0.xdc`

## :heavy_check_mark: Edit Localparams
The verilog file [`rtl/ddr3_controller`](https://github.com/AngeloJacobo/DDR3_Controller/blob/main/rtl/ddr3_controller.v) contains the timing parameters that needs to be configured by the user to align with the DDR3 device. User should base the timing values on _Chapter 13 Electrical Characteristics and AC Timing_ from [JEDEC DDR3 doc (page 169)](https://www.jedec.org/sites/default/files/docs/JESD79-3F.pdf). _The default values on the verilog file should generally work for DDR3-800_. 

### Note:  
[1]: For x16 DDR3 like in Arty S7, use `DQ_BITS` of 8 and `LANES` of 2 (not `DQ_BITS` of 16 or else the controller will not calibrate each bytes separately).  
[2]: The auxiliary line is intended for AXI-interface compatibility but is also utilized in the reset sequence, which is the origin of the minimum required width of 4.  
[3]: ODELAYE2 is supported if DDR3 device is connected to an HP (High-Powered) bank of FPGA. HR (High-Rank) bank does not support ODELAYE2 as based on [UG471 7-Series Select Guide (page 134)](https://docs.xilinx.com/v/u/en-US/ug471_7Series_SelectIO).   
[4]: This is the open-sourced [10Gb Ethernet Project](https://github.com/ZipCPU/eth10g).


***

#  Lint and Formal Verification
The easiest way to compile, lint, and formally verify the design is to run [`./run_compile.sh`](https://github.com/AngeloJacobo/DDR3_Controller/blob/main/run_compile.sh) on the top-level directory. This will first run [Verilator](https://verilator.org/guide/latest/install.html) lint. Most likely this will show errors:
> %Error: rtl//ddr3_phy: Cannot find file containing module:

Disregard these errors as Verilator cannot access the verilog files for Xilinx-exclusive IPs. Other than this kind of error, there should be no other errors or warning.

Next is compilation with [Yosys](https://github.com/YosysHQ/yosys), this will show warnings: 
> Warning: Replacing memory ... with list of registers.

Disregards this kind of warning as it just converts small memory elements in the design into a series of register elements.

After Yosys compilation is [Icarus Verilog](https://github.com/steveicarus/iverilog) compilation, this should not show any warning or errors but will display the `Test Functions` to verify that the verilog-functions return the correct values, and `Controller Parameters` to verify the top-level parameters are set properly. Delay values for some timing parameters are also shown.

Last is the [Symbiyosys Formal Verification](https://symbiyosys.readthedocs.io/en/latest/install.html), this will run [`ddr3.sby`](https://github.com/AngeloJacobo/DDR3_Controller/blob/main/ddr3.sby). These will run multiple verification tasks and will take some time (running each task might take 10 mins so overall it might take 1 and a half hours). A summary is shown at the end where all tasks passed:  

![image](https://github.com/AngeloJacobo/DDR3_Controller/assets/87559347/de554a92-880c-4513-83ba-a096da682f3b)


# Simulation

For simulation, the DDR3 SDRAM Verilog [Model from Micron](https://www.micron.com/search-results?searchRequest=%7B%22term%22%3A%22DDR3%20model%22%7D) is used. Import all simulation files under [./testbench](https://github.com/AngeloJacobo/DDR3_Controller/tree/main/testbench) to Vivado. [`ddr3_dimm_micron_sim.sv`](https://github.com/AngeloJacobo/DDR3_Controller/blob/main/testbench/ddr3_dimm_micron_sim.sv) is the top-level module which instantiates both the DDR3 memory controller and the Micron DDR3 model. This module issues read and write requests to the controller via the wishbone bus, then the returned data from read requests are verified if it matches the data written. Both sequential and random accesses are tested.

Currently, there are 2 general options for running the simulation and is defined by a `define` directive on the `ddr3_dimm_micron_sim.sv` file: `TWO_LANES_x8` and `EIGHT_LANES_x8`. `TWO_LANES_x8` simulates an Arty-S7 FPGA board which has an x16 DDR3, meanwhile `EIGHT_LANES_x8` simulates 8-lanes of x8 DDR3 module. **Make sure to change the organization via a `define` directive under [ddr3.sv](https://github.com/AngeloJacobo/DDR3_Controller/blob/main/testbench/ddr3.sv)** (`TWO_LANES_x8` must use `define x8` while `EIGHT_LANES_x8` must use `define x16`).


After configuring, run simulation. The `ddr3_dimm_micron_sim_behav.wcfg` contains the waveform. Shown below are the clocks:    
![image](https://github.com/AngeloJacobo/DDR3_Controller/assets/87559347/f11afd00-ea17-4669-bebb-9f22e8ae6f6d)  


As shown below, `command_used` shows the command issued at a specific time. During reads the `dqs` should toggle and `dq` should have a valid value, else they must be in high-impedance `Z`. Precharge and activate also happens between reads when row addresses are different. 
![image](https://github.com/AngeloJacobo/DDR3_Controller/assets/87559347/a289066f-2a5c-4d08-9660-a76cf537383a)


A part of internal test is to do alternate write then read consecutively as shown below. The data written must match the data read. `dqs` should also toggle along with the data written and read.  
![image](https://github.com/AngeloJacobo/DDR3_Controller/assets/87559347/817124fa-43d0-4e9f-94c4-2889614d7c87)


There are counters for the number of correct and wrong read data during the internal read/write test: `correct_read_data` and `wrong_read_data`. As shown below, the `wrong_read_data` must remain zero while `correct_read_data` must increment until it reaches the maximum (3499 on this example).  
![image](https://github.com/AngeloJacobo/DDR3_Controller/assets/87559347/06d7b4c0-cd40-4fd1-9bc3-6329237e46e3)

The simulation also reports the status of the simulation. For example, the report below:
> [10000 ps]  RD @ (0,   840) -> [10000 ps]  RD @ (0,   848) -> [10000 ps]  RD @ (0,   856) -> [10000 ps]  RD @ (0,   864) -> [10000 ps]  RD @ (0,   872) -> 

The format is [`time_delay`] `command` @ (`bank`, `address`), so `[10000 ps]  RD @ (0,   840)` means 10000 ps delay before a read command with bank 0 and address 840. Notice how each read command has a delay of 10000 ps or 10 ns from each other, since this has a controller clock of 100 MHz (10 ns clock period) this shows that there are no interruptions between sequential read commands resulting in a very high throughput. 

A short report is also shown in each test section:  
> DONE TEST 1: LAST ROW  
Number of Operations: 2304  
Time Started: 363390 ns  
Time Done: 387980 ns  
Average Rate: 10 ns/request  


This report is after a burst write then burst read. This report means there were 2304 write and read operation, and the average time per request is 10 ns (1 controller clock period of 100 MHz). The average rate is optimal since this is a burst write and read. But for random writes and reads:  
> DONE TEST 2: RANDOM  
Number of Operations: 2304  
Time Started: 387980 ns  
Time Done: 497660 ns  
Average Rate: 47 ns/request  


Notice how the average rate increased to 47 ns/request. Random access requires occasional precharge and activate which takes time and thus prolong the time for every read or write access. At the very end of the report shows a summary:

> TEST CALIBRATION  
[-]: write_test_address_counter = 5000  
[-]: read_test_address_counter = 2000  
[-]: correct_read_data = 3499  
[-]: wrong_read_data = 0  

> ------- SUMMARY -------  
Number of Writes = 4608  
Number of Reads = 4608  
Number of Success = 4604  
Number of Fails = 4  
Number of Injected Errors = 4  

The summary under `TEST CALIBRATION` are the results from the **internal** read/write test as part of the internal calibration. These are the same counters on the waveform shown before where the `wrong_read_data` should be zero. Under `SUMMARY` is the report from the **external** read/write test where the top-level simulation file `ddr3_dimm_micron_sim.sv` sends read/write request to the DDR3 controller via the wishbone bus. Notice that the number of fails (4) matches the number of injected errors (4) which is only proper.  

# Sample Projects
- The [Arty-S7](https://github.com/AngeloJacobo/DDR3_Controller/tree/main/arty_s7) project is a basic project for testing the DDR3 controller. The gist is that the 4 LEDS should light-up which means reset sequence is done and all internal read/write test passed during calibration. This project also uses a UART line, sending small letters via UART will write those corresponding small letters to memory, meanwhile sending capital letters will read those small letters back from memory. To run this project on your Arty-S7 board, import all verilog files and xdc file under `arty_s7/` including the verilog files under the submodule `verilog-uart/rtl/`. Instantiate a clock wizard with the following settings:  

![image](https://github.com/AngeloJacobo/DDR3_Controller/assets/87559347/b35cecb4-c8cc-4b0f-93ff-8fb2ef337dde)

This will run the DDR3 controller at 333 MHz (3 ns clock period) which is the [maximum clock period for Arty-S7](https://digilent.com/reference/programmable-logic/arty-s7/reference-manual). Upload the bitstream to Arty-S7, after around 2 seconds the 4 LEDS should light up.

- The [10Gb Ethernet Switch](https://github.com/ZipCPU/eth10g) project utilizes this DDR3 controller for accessing a single-rank DDR3 module (8 lanes of x8 DDR3) at DDR3-800 (100 MHz controller and 400 MHz PHY).

# Other Open-Sourced DDR3 Controllers
(soon...)
