# Overview
This is Original Spec file , which convert from original

Next:
will will break down it to 3 level, each level has one or more markdown depends on if it's too huge
ex
spec_kg-h1.md
spec_kg-h1-1.md

---
# file: spec_kg.ori_doc.pdf

## outline
1. **Introduction**
   - 1.1 Features
2. **IO Ports**
   - 2.1 Core Parameters
     - 2.1.1 ARST_LVL
   - 2.2 WISHBONE Interface Signals
   - 2.3 External Connections
3. **Registers**
   - 3.1 Registers List
   - 3.2 Register Description
     - 3.2.1 Prescale Register
     - 3.2.2 Control Register
     - 3.2.3 Transmit Register
     - 3.2.4 Receive Register
     - 3.2.5 Command Register
     - 3.2.6 Status Register
4. **Operation**
   - 4.1 System Configuration
   - 4.2 I2C Protocol
     - 4.2.1 START Signal
     - 4.2.2 Slave Address Transfer
     - 4.2.3 Data Transfer
     - 4.2.4 STOP Signal
   - 4.3 Arbitration Procedure
     - 4.3.1 Clock Synchronization
     - 4.3.2 Clock Stretching
5. **Architecture**
   - 5.1 Clock Generator
   - 5.2 Byte Command Controller
   - 5.3 Bit Command Controller
   - 5.4 DataIO Shift Register
6. **Programming Examples**
   - Example 1: Write 1 Byte of Data to a Slave
   - Example 2: Read a Byte of Data from an I2C Memory Device
7. **Appendix A: Synthesis Results**

 
# 1. Introduction

## 1.1 Overview

I2C is a two-wire, bi-directional serial bus that provides a simple and efficient method of data exchange between devices. It is most suitable for applications requiring occasional communication over a short distance between many devices. The I2C standard is a true multi-master bus including collision detection and arbitration that prevents data corruption if two or more masters attempt to control the bus simultaneously.

The interface defines three transmission speeds:

- **Normal**: 100 Kbps
- **Fast**: 400 Kbps
- **High-Speed**: 3.5 Mbps

Only 100 Kbps and 400 Kbps modes are supported directly. For high-speed mode, special IOs are needed. If these IOs are available and used, then high-speed mode is also supported.

## 1.2 Features

- Compatible with Philips I2C standard.
- Multi-master operation.
- Software-programmable clock frequency.
- Clock stretching and wait-state generation.
- Software-programmable acknowledge bit.
- Interrupt or bit-polling driven byte-by-byte data transfers.
- Arbitration lost interrupt with automatic transfer cancellation.
- Start/Stop/Repeated Start/Acknowledge generation.
- Start/Stop/Repeated Start detection.
- Bus busy detection.
- Supports 7-bit and 10-bit addressing modes.
- Operates from a wide range of input clock frequencies.
- Static synchronous design.
- Fully synthesizable.
# 2. IO Ports

## 2.1 Core Parameters

| **Parameter** | **Type** | **Default** | **Description**                                                                            |
| ------------- | -------- | ----------- | ------------------------------------------------------------------------------------------ |
| `ARST_LVL`    | Bit      | `1'b0`      | Asynchronous reset level can be set to either active high (`1'b1`) or active low (`1'b0`). |
## 2.2 WISHBONE Interface Signals

| **Port**    | **Width** | **Direction** | **Description**                              |
| ----------- | --------- | ------------- | -------------------------------------------- |
| `wb_clk_i`  | 1         | Input         | Master clock.                                |
| `wb_rst_i`  | 1         | Input         | Synchronous reset, active high.              |
| `arst_i`    | 1         | Input         | Asynchronous reset.                          |
| `wb_adr_i`  | 3         | Input         | Lower address bits.                          |
| `wb_dat_i`  | 8         | Input         | Data input to the core.                      |
| `wb_dat_o`  | 8         | Output        | Data output from the core.                   |
| `wb_we_i`   | 1         | Input         | Write enable signal.                         |
| `wb_stb_i`  | 1         | Input         | Strobe signal/core select input.             |
| `wb_cyc_i`  | 1         | Input         | Valid bus cycle input signal.                |
| `wb_ack_o`  | 1         | Output        | Acknowledge signal for bus cycle completion. |
| `wb_inta_o` | 1         | Output        | Interrupt signal output.                     |

The core features a WISHBONE RevB.3 compliant WISHBONE Classic interface. All output signals are registered. Each access takes 2 clock cycles.  
`arst_i` is not a WISHBONE-compatible signal. It is provided for FPGA implementations. Using `arst_i` instead of `wb_rst_i` can result in lower cell usage and higher performance, as most FPGAs provide a dedicated asynchronous reset path. Use either `arst_i` or `wb_rst_i` and tie the other to a negated state.

## 2.3 External Connections

| **Port**     | **Width** | **Direction** | **Description**                          |
| ------------ | --------- | ------------- | ---------------------------------------- |
| `scl_pad_i`  | 1         | Input         | Serial Clock (SCL) line input.           |
| `scl_pad_o`  | 1         | Output        | Serial Clock (SCL) line output.          |
| `scl_pad_oe` | 1         | Output        | Serial Clock (SCL) output enable signal. |
| `sda_pad_i`  | 1         | Input         | Serial Data (SDA) line input.            |
| `sda_pad_o`  | 1         | Output        | Serial Data (SDA) line output.           |
| `sda_pad_oe` | 1         | Output        | Serial Data (SDA) output enable signal.  |

The I2C interface uses a serial data line (SDA) and a serial clock line (SCL) for data transfers. All devices connected to these two signals must have open drain or open collector outputs. Both lines must be pulled up to VCC by external resistors.

The tri-state buffers for the SCL and SDA lines must be added at a higher hierarchical level. Connections should be made according to the following figure:

---
#### Figure. 2.3 tri-state buffers
![[figure.2.3.png]]
Describe Figure 2.3

This figure shows the tri-state buffer configuration for the I2C **SCL** (Serial Clock) and **SDA** (Serial Data) lines. Each line has:

1. **Input Signal (`scl_pad_i`/`sda_pad_i`)**: Reads the current state of the line.
2. **Output Signal (`scl_pad_o`/`sda_pad_o`)**: Drives the line with the core's signal.
3. **Output Enable Signal (`scl_padoen_o`/`sda_padoen_o`)**: Controls whether the core actively drives the line (low) or releases it to a high-impedance state (high).

This ensures safe communication on a shared I2C bus by allowing only one device to drive the lines at a time, while others monitor or release them.

---
#### Verilog Code for Buffer Assignment:
```verilog
assign scl = scl_pad_oe ? 1'bz : scl_pad_o;
assign sda = sda_pad_oe ? 1'bz : sda_pad_o;
assign scl_pad_i = scl;
assign sda_pad_i = sda; 
```

This configuration ensures proper signal handling for bidirectional data communication in compliance with the I2C protocol.


# 3. Registers

The I2C core uses several registers for configuration, status monitoring, and data transfer. Each register serves a specific purpose and is accessed using its unique address.


## 3.1 Registers List

| **Name** | **Address** | **Width** | **Access** | **Description**                    |
| -------- | ----------- | --------- | ---------- | ---------------------------------- |
| `PRERlo` | 0x00        | 8         | RW         | Clock prescale register low byte.  |
| `PRERhi` | 0x01        | 8         | RW         | Clock prescale register high byte. |
| `CTR`    | 0x02        | 8         | RW         | Control register.                  |
| `TXR`    | 0x03        | 8         | W          | Transmit register.                 |
| `RXR`    | 0x03        | 8         | R          | Receive register.                  |
| `CR`     | 0x04        | 8         | W          | Command register.                  |
| `SR`     | 0x04        | 8         | R          | Status register.                   |

---


## 3.2 Register Description


### 3.2.1 Prescale Register

The Prescale Register is used to configure the clock prescaler, which determines the frequency of the SCL clock line. It consists of two 8-bit registers: `PRERlo` (low byte) and `PRERhi` (high byte).

---

| **Field**        | **Description**                                                                                                                                                                                                |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Purpose**      | Configures the prescaler to set the SCL clock frequency.                                                                                                                                                       |
| **Details**      | The I²C core uses an internal clock that operates at $5 \times \text{SCL}$. The prescale value is calculated using the formula: <br> $$ \text{prescale} = \frac{\text{wb\_clk\_i}}{5 \times \text{SCL}} - 1 $$ |
| **Restrictions** | Change the prescale register value only when the `EN` bit in the Control Register is cleared.                                                                                                                  |
| **Reset Value**  | `0xFFFF`                                                                                                                                                                                                       |

---

#### **Example Calculation**
- **Input Clock (`wb_clk_i`)**: 32 MHz  
- **Desired SCL Clock Frequency**: 100 kHz  
$$
\text{prescale} = \frac{32\ \text{MHz}}{5 \times 100\ \text{kHz}} - 1 = 63\ (\text{decimal}) = 3F\ (\text{hexadecimal})
$$

This calculated value (`0x3F`) is then written into the Prescale Register to achieve the desired clock frequency.

---

#### **Usage Notes**
- Both `PRERlo` and `PRERhi` are 8-bit wide. Together, they form the 16-bit prescaler value.
- The prescale value must be programmed carefully to ensure correct timing on the I²C bus.

The Prescale Register ensures accurate timing for the SCL clock, a critical factor for proper I²C communication.



### 3.2.2 Control Register

The Control Register (`CTR`) is used to enable or disable the I²C core and its interrupt functionality.

| **Bit #** | **Access** | **Description**                                                                 |
|-----------|------------|---------------------------------------------------------------------------------|
| **7**     | RW         | **`EN`**, I²C core enable bit. <br> - `1`: Core is enabled. <br> - `0`: Core is disabled. |
| **6**     | RW         | **`IEN`**, I²C interrupt enable bit. <br> - `1`: Interrupts are enabled. <br> - `0`: Interrupts are disabled. |
| **5:0**   | RW         | Reserved.                                                                       |


#### **Details**
- The `EN` bit must be set to `1` to activate the I²C core and allow it to respond to commands. When set to `0`, the core is disabled, and ongoing transfers are halted.
- The `IEN` bit enables interrupt-driven communication. If `IEN` is set to `1`, the `wb_inta_o` signal is asserted when the `IF` flag in the Status Register is set.

#### **Reset Value**
- **`0x00`**

#### **Usage Notes**
- Clear the `EN` bit only when no transfer is in progress. For example:
  - After a STOP command.
  - When the Command Register (`CR`) has the `STO` bit set.
- Clearing the `EN` bit during an active transfer may hang the I²C bus.

This register is crucial for managing the I²C core's operational state and enabling interrupt-driven data transfers.


### 3.2.3 Transmit Register

| **Bit #** | **Access** | **Description**                                                                                                                                                                        |
| --------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **7:1**   | W          | Next byte to transmit via I²C.                                                                                                                                                         |
| **0**     | W          | In case of a data transfer, this bit represents the data's LSB. In case of a slave address transfer, this bit represents the RW bit: - `1`: Reading from slave - `0`: Writing to slave |

**Reset Value**: `0x00`


### 3.2.4 Receive Register

| **Bit #** | **Access** | **Description**             |
| --------- | ---------- | --------------------------- |
| **7:0**   | R          | Last byte received via I²C. |

**Reset Value**: `0x00`

### 3.2.5 Command Register

| **Bit #** | **Access** | **Description**                                                      |
| --------- | ---------- | -------------------------------------------------------------------- |
| **7**     | W          | `STA`: Generate (repeated) START condition.                          |
| **6**     | W          | `STO`: Generate STOP condition.                                      |
| **5**     | W          | `RD`: Read from slave.                                               |
| **4**     | W          | `WR`: Write to slave.                                                |
| **3**     | W          | `ACK`: When a receiver, send ACK (`0`) or NACK (`1`).                |
| **2:1**   | W          | Reserved.                                                            |
| **0**     | W          | `IACK`: Interrupt acknowledge. When set, clears a pending interrupt. |

**Reset Value**: `0x00`

**Note**:  
The `STA`, `STO`, `RD`, `WR`, and `IACK` bits are cleared automatically. These bits are always read as zeros.

### 3.2.6 Status Register

The Status Register provides information about the current state of the I²C core and the bus. It is read-only and reflects the results of previous operations or the current status of ongoing ones.

|**Bit #**|**Access**|**Description**|
|---|---|---|
|**7**|R|`RxACK`: Indicates whether the slave sent an ACK. - `0`: Acknowledge received - `1`: No acknowledge received|
|**6**|R|`BUSY`: Indicates whether the I²C bus is busy. - `1`: Bus is busy - `0`: Bus is idle|
|**5**|R|`AL`: Arbitration lost. Set when the core loses arbitration on the bus.|
|**4:2**|Reserved||
|**1**|R|`TIP`: Transfer in progress. - `1`: Data transfer is ongoing - `0`: Data transfer is complete|
|**0**|R|`IF`: Interrupt flag. Set when an interrupt is pending.|

**Reset Value**: `0x00`

---

### Additional Notes on the Registers

- The core responds to new commands only when the `EN` bit in the Control Register is set.
- The `STA`, `STO`, `RD`, `WR`, and `IACK` bits in the Command Register are automatically cleared after execution.
- Clearing the `EN` bit while a transfer is in progress may cause the core to hang the I²C bus.
- Always check the `TIP` bit before issuing new commands to ensure the current operation is complete.

This concludes the description of the I²C registers, which are critical for controlling, monitoring, and executing operations on the I²C bus.


---


# 4. Operation

## 4.1 System Configuration

The I²C system uses a serial data line (SDA) and a serial clock line (SCL) for data transfers. All devices connected to these two signals must have open-drain or open-collector outputs. The logic AND function is exercised on both lines with external pull-up resistors.

Data is transferred between a Master and a Slave synchronously to SCL on the SDA line on a byte-by-byte basis. Each data byte is 8 bits long. There is one SCL clock pulse for each data bit with the MSB being transmitted first. An acknowledge bit follows each transferred byte. Each bit is sampled during the high period of SCL; therefore, the SDA line may be changed only during the low period of SCL and must be held stable during the high period of SCL. A transition on the SDA line while SCL is high is interpreted as a command (see START and STOP signals).

---

## 4.2 I²C Protocol

Normally, a standard communication consists of four parts:
1. START signal generation
2. Slave address transfer
3. Data transfer
4. STOP signal generation
---
#### Figure. 4.2 SCL / SDA protocol weaveform
![[figure.4.2.png]]

Describe Figure 4.2

This figure illustrates the timing diagram of a typical I²C data transfer sequence, showcasing the relationship between the Serial Clock Line (**SCL**) and Serial Data Line (**SDA**) during a transmission.


Explanation of the Signals

1. SCL (Serial Clock Line):
- The clock signal synchronizes data transfer on the I²C bus.
- Each data bit on the SDA line is valid during the high period of SCL.

 2. SDA (Serial Data Line):
- The data signal transmits the actual information, including address and data bits.
- Changes on SDA occur during the low period of SCL to ensure stability during the high period.


Phases of the Data Transfer

1. **START Condition (`S`):**
   - The transmission begins with a START signal.
   - Defined as a **high-to-low transition of SDA while SCL is high**.

2. **Address Transfer (`A7` to `A1`):**
   - The master transmits a **7-bit slave address** (`A7` to `A1`) to identify the target device on the bus.

3. **Read/Write Bit (`RW`):**
   - The 8th bit indicates the operation type:
     - `0`: Write operation.
     - `1`: Read operation.

4. **Acknowledge Bit (`ACK`):**
   - The addressed slave device responds by pulling SDA low during the 9th clock cycle, indicating it is ready to communicate.

5. **Data Byte Transfer (`D7` to `D0`):**
   - The actual data is transmitted in an **8-bit byte** (`D7` to `D0`), with the most significant bit (MSB) first.
   - The slave or master acknowledges the receipt of each byte by sending an ACK bit.

6. **No Acknowledge (`NACK`):**
   - If the master sends a NACK after receiving the last byte, it signals the end of the read operation.

7. **STOP Condition (`P`):**
   - The transmission ends with a STOP signal.
   - Defined as a **low-to-high transition of SDA while SCL is high**.


 **Key Observations**
- **Stable Data During SCL High**: The SDA line remains stable (unchanged) when SCL is high. Any change on SDA during this time indicates a START or STOP condition.
- **Bit-by-Bit Acknowledgment**: Each 8-bit transmission (address or data) is followed by an acknowledgment (ACK or NACK) bit.
- **MSB First**: Data is transmitted starting with the most significant bit.

This timing diagram represents the core operational sequence for communication on the I²C bus, ensuring synchronization and data integrity between devices.

---
### 4.2.1 START Signal

When the bus is free/idle, meaning no master device is engaging the bus (both SCL and SDA lines are high), a master can initiate a transfer by sending a START signal. A START signal, usually referred to as the **S-bit**, is defined as a **high-to-low transition of SDA while SCL is high**. The START signal denotes the beginning of a new data transfer.  

A **Repeated START** is a START signal without first generating a STOP signal. The master uses this method to communicate with another slave or the same slave in a different transfer direction (e.g., from writing to a device to reading from a device) without releasing the bus.

The core generates a START signal when the `STA` bit in the Command Register is set and the `RD` or `WR` bits are set. Depending on the current status of the SCL line, a START or Repeated START is generated.

---

### 4.2.2 Slave Address Transfer

The first byte of data transferred by the master immediately after the START signal is the slave address. This is a **7-bit calling address** followed by a **Read/Write (RW) bit**. The RW bit signals the slave about the data transfer direction:
- `0` = Write operation
- `1` = Read operation

No two slaves in the system can have the same address. Only the slave with an address matching the one transmitted by the master will respond by returning an **ACK bit** (pulling the SDA line low at the 9th SCL clock cycle).

**Note**: The core supports 10-bit slave addresses by generating two address transfers. See the Philips I²C specifications for more details.

The core treats a Slave Address Transfer as any other write action. Store the slave device’s address in the Transmit Register and set the `WR` bit. The core will then transfer the slave address on the bus.

---

### 4.2.3 Data Transfer

Once successful slave addressing has been achieved, the data transfer can proceed on a byte-by-byte basis in the direction specified by the RW bit sent by the master. Each transferred byte is followed by an acknowledge bit on the 9th SCL clock cycle. If the slave signals a No Acknowledge (**NACK**), the master can:
- Generate a STOP signal to abort the data transfer.
- Generate a Repeated START signal to start a new transfer cycle.

If the master, as the receiving device, does not acknowledge the slave, the slave releases the SDA line for the master to generate a STOP or Repeated START signal.

**Writing Data to a Slave**:
- Store the data to be transmitted in the Transmit Register.
- Set the `WR` bit.

**Reading Data from a Slave**:
- Set the `RD` bit.

During a transfer, the core sets the `TIP` flag, indicating that a Transfer is In Progress. When the transfer is done, the `TIP` flag is reset, the `IF` flag is set, and, when enabled, an interrupt is generated. The Receive Register contains valid data after the `IF` flag has been set. The user may issue a new write or read command when the `TIP` flag is reset.

---

### 4.2.4 STOP Signal

The master can terminate the communication by generating a STOP signal. A STOP signal, usually referred to as the **P-bit**, is defined as a **low-to-high transition of SDA while SCL is high**.

---

## 4.3 Arbitration Procedure

### 4.3.1 Clock Synchronization

The I²C bus is a true multi-master bus that allows more than one master to be connected to it. If two or more masters simultaneously try to control the bus, a **clock synchronization procedure** determines the bus clock.  

Because of the wired-AND connection of the I²C signals, a high-to-low transition affects all devices connected to the bus. A high-to-low transition on the SCL line causes all concerned devices to count off their low period. Once a device clock has gone low, it will hold the SCL line in that state until its clock high state is reached. The SCL line will be held low by the device with the longest low period, and high by the device with the shortest high period.

---
#### Figure 4.3.1 SCL multi-master  clock synchronize

![[figure.4.3.1.png]]


Describe Figure 4.3.1

This figure illustrates the **clock synchronization** process on the I²C bus when multiple masters (e.g., Master1 and Master2) are attempting to drive the clock line (SCL) simultaneously.

Key Points of the Diagram

1. **SCL1 and SCL2**:
    
    - Represent the clock signals generated by two different masters (Master1 and Master2).
    - Each master attempts to control the clock's high and low periods.
2. **Wired-AND SCL**:
    
    - The actual SCL line is the result of the **wired-AND** logic combining the clock signals from all masters.
    - In a wired-AND configuration:
        - If one master drives SCL low, the line remains low (dominant state).
        - The line goes high only when all masters release it (i.e., no master is driving it low).
3. **Clock Synchronization**:
    
    - **Low Period**:
        - The SCL line remains low until **all masters finish their low periods**. The longest low period determines the total low time.
    - **High Period**:
        - The high period begins when all masters release SCL to high. The shortest high period determines when the clock will transition back to low.
4. **Wait State**:
    
    - A **wait state** occurs during the transition from the low period to the high period. The wait state ensures that the SCL line remains synchronized across all masters before resuming operation.


**Explanation of Behavior**

- In an I²C multi-master setup, clock synchronization is necessary to ensure proper communication timing and avoid conflicts.
- The **master with the longest low period** effectively sets the pace for the low phase of the clock.
- The **master with the shortest high period** determines the duration of the high phase of the clock.

---
### 4.3.2 Clock Stretching

Slave devices can use the clock synchronization mechanism to slow down the transfer bit rate. After the master has driven SCL low, the slave can drive SCL low for the required period and then release it. If the slave’s SCL low period is greater than the master’s SCL low period, the resulting SCL bus signal low period is stretched, inserting wait states.

# 5. Architecture

## 5.0 Overview

The I²C core is designed with a modular architecture, comprising key functional blocks that interact to manage the I²C protocol's operations efficiently. This chapter outlines the core components and their functions.


**Core Structure**

The I²C core is built around four primary blocks:

1. Clock Generator
2. Byte Command Controller
3. Bit Command Controller
4. DataIO Shift Register

Other auxiliary blocks are used for interfacing and storing temporary values.

---
#### Figure. 5.1 Internal structure I2C Master Core

![[figure.5.1.png]]


Describe Figure. 5.1

This diagram represents the **block architecture** of the I²C core, showcasing its key functional components and their interactions.

**Key Components**

1. **WISHBONE Interface**:
   - Acts as the communication bridge between the external system and the I²C core.
   - Provides access to internal registers and facilitates data exchange.

2. **Registers**:
   - **Prescale Register**: Configures the SCL clock frequency by working with the Clock Generator.
   - **Command Register**: Receives high-level commands such as START, STOP, READ, and WRITE from the master.
   - **Status Register**: Reflects the current state of the I²C bus and the core (e.g., busy, interrupt flags, arbitration lost).
   - **Transmit Register**: Temporarily holds the data to be transmitted on the I²C bus.
   - **Receive Register**: Temporarily stores data received from the I²C bus.

3. **Clock Generator**:
   - Derives the internal clock signals required for I²C operations based on the prescaler value.
   - Handles clock stretching, allowing slower devices to delay data transfer.

4. **Byte Command Controller**:
   - Processes commands from the Command Register and converts them into sequences of single-byte operations.
   - Orchestrates the data flow between the DataIO Shift Register and the Bit Command Controller.

5. **Bit Command Controller**:
   - Handles the low-level signal generation for the SDA and SCL lines.
   - Executes bit-level operations like START, STOP, data transfer, and acknowledgment.

6. **DataIO Shift Register**:
   - Acts as a temporary buffer for data transfers.
   - **Transmit Path**: Transfers data from the Transmit Register to the Bit Command Controller.
   - **Receive Path**: Transfers data from the Bit Command Controller to the Receive Register.

Data Flow
- The **WISHBONE Interface** provides access to the registers.
- The **Prescale Register** configures the **Clock Generator**, which provides clock signals to the core.
- The **Command Register** and **Byte Command Controller** coordinate with the **Bit Command Controller** to execute commands on the SDA and SCL lines.
- Data is moved between the **Transmit Register**, **Receive Register**, and the I²C bus via the **DataIO Shift Register**.
---
## 5.1 Clock Generator

The Clock Generator provides the clock enable signal required for the operation of the Bit Command Controller. It generates a signal at internal $$ 4 \times \text{Fscl} $$ clock enable signal, ensuring synchronization with the I²C protocol's timing requirements. The Clock Generator also supports ==clock stretching, allowing slower slave devices== to hold the clock line low, effectively pausing the communication until they are ready.
## 5.2 Byte Command Controller

The Byte Command Controller handles the I²C traffic at the byte level. It interprets the commands issued in the Command Register and translates them into sequences for single-byte operations. The controller takes care of generating and detecting the START, STOP, and Repeated START conditions. It also sequences each byte operation into individual bit-level operations and passes them to the Bit Command Controller for execution.

---
#### Figure 5.2 FSM
![[figure.5.2.png]]
**Describe Figure 4.2**

This flowchart represents the **state machine** of the I²C core, detailing the transitions between different operational states for reading, writing, and acknowledgment. Here’s the explanation of each part:


**States and Transitions**

1. **Idle State**:
   - The system starts in the idle state.
   - If no operation (Read/Write bit) is set, the system remains in this state.

2. **Read/Write Bit Check**:
   - If the Read/Write bit is set, the system transitions to the next state.
   - Otherwise, it loops back to the idle state.

3. **START Bit Check**:
   - If the START bit is set, the system moves to the START signal state.
   - If not, it remains in the current state.

4. **START Signal State**:
   - This state ensures that a START signal is properly generated.
   - The system checks if the START condition has been successfully generated.

5. **START Generated**:
   - If the START condition is generated:
     - The system transitions to either the READ or WRITE state, depending on the operation type.
   - If not, it stays in the START signal state.

6. **READ State**:
   - In this state, the system attempts to read a byte of data.
   - It checks if the byte has been successfully read.

7. **WRITE State**:
   - In this state, the system attempts to write a byte of data.
   - It checks if the byte has been successfully written.

8. **ACK State**:
   - After a byte is read or written, the system moves to the acknowledgment state.
   - It checks if the ACK bit has been received or written correctly.

9. **ACK Bit Check**:
   - If the ACK bit is correctly handled, the system loops back to the idle state or continues with the next operation.

**Key Points**
- The flowchart ensures proper sequencing of I²C operations, including reading, writing, and acknowledgment handling.
- Each transition is dependent on specific conditions, such as bit settings or the successful generation of START, STOP, or ACK signals.
- If any condition is not met, the system remains in the current state until resolved.

This state machine provides a structured approach for managing I²C communication, ensuring reliable data transfer between devices.

---
## 5.3 Bit Command Controller


The Bit Command Controller handles the actual transmission of data and the generation of the specific levels for START, Repeated START, and STOP signals by controlling the SCL and SDA lines. The Byte Command Controller tells the Bit Command Controller which operation has to be performed. For a single byte read, the Bit Command Controller receives 8 separate read commands. Each bit operation is divided into 5 pieces (idle and A, B, C, and D), except for a STOP operation which is divided into 4 pieces (idle and A, B, and C).

#### Figure 5.3 bit options FSM explain in  waveform

![[figure.5.3.png]]

Describe Figure 5.3
This figure illustrates the detailed **bit-level operations** of the I²C protocol, highlighting how the **SCL** (clock line) and **SDA** (data line) signals transition during various operations: START, Repeated START, STOP, WRITE, and READ. Each operation is divided into phases labeled **A**, **B**, **C**, and **D** for precise control.

**Operations Explained**

1. **START Condition (Start)**:
   - **SCL**: Stable high during the beginning.
   - **SDA**: Transitions from high to low while **SCL** remains high, initiating communication.

2. **Repeated START Condition (Rep Start)**:
   - **SCL**: Stable high during the repeated start.
   - **SDA**: Similar to the START condition, transitions from high to low while **SCL** is high, indicating a repeated transfer without a STOP condition.

3. **STOP Condition (Stop)**:
   - **SCL**: Stable high during the stop sequence.
   - **SDA**: Transitions from low to high while **SCL** remains high, terminating communication.

4. **WRITE Operation**:
   - **SCL**: Pulses for each bit of data (phases A, B, C, and D represent bit-level timing).
   - **SDA**: Holds each data bit stable during the high phase of **SCL**.

5. **READ Operation**:
   - **SCL**: Pulses for each bit of data.
   - **SDA**: Driven by the slave device. Data is read by the master during the high phase of **SCL**.


 **Phases A, B, C, D**
- **A**: Initial state or idle state before a transition.
- **B**: Rising edge of the clock where data is sampled.
- **C**: Stable high phase where data is held.
- **D**: Falling edge of the clock, completing the bit transfer.


This figure represents the precise timing and signal transitions required to perform each operation on the I²C bus, ensuring protocol compliance and proper communication between devices.

---
## 5.4 DataIO Shift Register

The DataIO Shift Register contains the data associated with the current transfer. During a read action, data is shifted in from the SDA line. After a byte has been read the contents are copied into the Receive Register. During a write action, the Transmit Register’s contents are copied into the DataIO Shift Register and are then transmitted onto the SDA line.