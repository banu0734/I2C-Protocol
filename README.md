# I2C Master Controller Design and Verification Using SystemVerilog

---

## 📌 Introduction

This project implements and verifies an **I2C Master Controller** using Verilog and SystemVerilog.  
A class-based verification environment is used to validate protocol behavior under multiple transaction modes.

---

## 🎯 Objectives

- Design FSM-based I2C Master  
- Verify Read and Write Operations  
- Implement Multiple Drivers  
- Perform Protocol Verification  
- Analyze Simulation Waveforms  

---

## ✨ Features

- FSM-Based I2C Controller  
- Multiple Verification Modes  
- Randomized Testing  
- Protocol-Level Drivers  
- Waveform-Based Debugging  

---

# 📂 Project Architecture

| Block | Description |
|-------|-------------|
| RTL | I2C Master |
| Interface | DUT Interface |
| Packet | Transaction |
| Generator | Stimulus |
| Driver | Protocol |
| Testbench | Top |

---

# 🔧 RTL Design (i2c_master.v)

```verilog
// ================= I2C MASTER =================

module i2c_master(
  addr_in, data_in, write, clk_in, reset_in,
  sda_in, i2c_sda_out, i2c_scl_out, read_out
);

input clk_in, reset_in, write, sda_in;
input [6:0] addr_in;
input [7:0] data_in;

output reg i2c_sda_out, i2c_scl_out;
output reg [7:0] read_out;

reg [2:0] acount, dcount;
reg sda_en, sda_out, scl_en;
reg [2:0] state, next_state;
reg [7:0] i2c_slave_in;

parameter IDLE=0, START=1, ADDRESS=2, READ_WRITE=3,
          ADDR_ACK=4, DATA=5, DATA_ACK=6, STOP=7;

// State Register
always @(negedge clk_in) begin
  if(reset_in) begin
    state <= IDLE;
    read_out <= 0;
    i2c_slave_in <= 0;
  end
  else begin
    state <= next_state;
    if(state==ADDRESS) acount<=acount-1;
    else if(state==DATA) dcount<=dcount-1;
  end
end

// FSM
always @(*) begin
  case(state)

  IDLE: begin
    scl_en=0; sda_out=1; sda_en=1;
    acount=6; dcount=7;
    if(addr_in!=0) next_state=START;
    else next_state=IDLE;
  end

  START: begin
    scl_en=0; sda_out=0; sda_en=1;
    next_state=ADDRESS;
  end

  ADDRESS: begin
    scl_en=1; sda_en=1;
    sda_out=addr_in[acount];
    if(acount!=0) next_state=ADDRESS;
    else next_state=READ_WRITE;
  end

  READ_WRITE: begin
    scl_en=1; sda_en=1;
    sda_out=write;
    next_state=ADDR_ACK;
  end

  ADDR_ACK: begin
    scl_en=1;
    if(sda_in==0) begin
      sda_en=0;
      next_state=DATA;
    end
    else next_state=STOP;
  end

  DATA: begin
    scl_en=1;

    if(write==0) begin
      sda_en=1;
      sda_out=data_in[dcount];
      if(dcount!=0) next_state=DATA;
      else next_state=DATA_ACK;
    end
    else begin
      sda_en=0;
      i2c_slave_in[dcount]=sda_in;
      if(dcount!=0) next_state=DATA;
      else next_state=DATA_ACK;
    end
  end

  DATA_ACK: begin
    scl_en=1;
    next_state=STOP;
  end

  STOP: begin
    read_out=i2c_slave_in;
    scl_en=0;
    sda_out=0;
    sda_en=1;
    next_state=IDLE;
  end

  endcase
end

assign i2c_scl_out = (scl_en)?clk_in:1'b1;
assign i2c_sda_out = (sda_en)?sda_out:sda_in;

endmodule
```

---

# 🔌 Interface (interface.sv)

```systemverilog
interface intf;
  logic clk_in, reset_in;
  logic [6:0] addr_in;
  logic write;
  logic [7:0] data_in;
  logic sda_in;
  logic i2c_sda_out, i2c_scl_out;
  logic [7:0] read_out;
endinterface
```

---

# 📦 Packet (packet.sv)

```systemverilog
class packet;
  rand logic [6:0] addr_in1;
  rand logic [7:0] data_in1;
  rand logic write1;
  logic sda_in1;
endclass
```

---

# ⚙ Generator (generator.sv)

```systemverilog
class generator;

  packet pkt;
  mailbox m;

  function new(mailbox m);
    this.m = m;
  endfunction

  task run;
    repeat(4) begin
      pkt = new();
      pkt.randomize();
      m.put(pkt);
      #5;
    end
  endtask

endclass
```

---

# 🚗 Driver Codes

---

## ✅ Count-Based Driver

```systemverilog
class driver_count;

  packet pkt;
  virtual intf i1;
  mailbox m1;
  int count;

  function new(mailbox m1, virtual intf i1);
    this.m1=m1;
    this.i1=i1;
    count=0;
  endfunction

  task run;
    repeat(4) begin
      m1.get(pkt);
      count++;

      i1.addr_in<=pkt.addr_in1;
      i1.data_in<=pkt.data_in1;

      if(count%2) begin
        i1.write<=0; write_cycle;
      end
      else begin
        i1.write<=1; read_cycle;
      end
    end
  endtask

  task write_cycle; start; addr; ack; data; ack; stop; endtask
  task read_cycle;  start; addr; ack; data; ack; stop; endtask

  task start; i1.sda_in=1; endtask
  task addr; repeat(7) @(negedge i1.i2c_scl_out); endtask
  task data; repeat(8) @(negedge i1.i2c_scl_out); endtask
  task ack; i1.sda_in=0; @(negedge i1.i2c_scl_out); i1.sda_in=1; endtask
  task stop; i1.sda_in=1; endtask

endclass
```

---

## ✅ Randomized Driver

```systemverilog
class driver_random;

  packet pkt;
  virtual intf i1;
  mailbox m1;

  function new(mailbox m1, virtual intf i1);
    this.m1=m1;
    this.i1=i1;
  endfunction

  task run;
    repeat(5) begin
      pkt=new();
      pkt.randomize();

      i1.addr_in<=pkt.addr_in1;
      i1.data_in<=pkt.data_in1;
      i1.write<=pkt.write1;

      protocol;
    end
  endtask

  task protocol; start; addr; ack; data; ack; stop; endtask

  task start; i1.sda_in=1; endtask
  task addr; repeat(7) @(negedge i1.i2c_scl_out); endtask
  task data; repeat(8) @(negedge i1.i2c_scl_out); endtask
  task ack; i1.sda_in=0; @(negedge i1.i2c_scl_out); i1.sda_in=1; endtask
  task stop; i1.sda_in=1; endtask

endclass
```

---

## ✅ Repeated Read Driver

```systemverilog
class driver_rread;

  packet pkt;
  virtual intf i1;
  mailbox m1;

  function new(mailbox m1, virtual intf i1);
    this.m1=m1;
    this.i1=i1;
  endfunction

  task run;
    repeat(4) begin
      m1.get(pkt);

      i1.addr_in<=pkt.addr_in1;
      i1.write<=1;

      read_cycle;
    end
  endtask

  task read_cycle; start; addr; ack; data; ack; stop; endtask

  task start; i1.sda_in=1; endtask
  task addr; repeat(7) @(negedge i1.i2c_scl_out); endtask
  task data; repeat(8) @(negedge i1.i2c_scl_out); endtask
  task ack; i1.sda_in=0; @(negedge i1.i2c_scl_out); i1.sda_in=1; endtask
  task stop; i1.sda_in=1; endtask

endclass
```

---

## ✅ Repeated Write Driver

```systemverilog
class driver_rwrite;

  packet pkt;
  virtual intf i1;
  mailbox m1;

  function new(mailbox m1, virtual intf i1);
    this.m1=m1;
    this.i1=i1;
  endfunction

  task run;
    repeat(4) begin
      m1.get(pkt);

      i1.addr_in<=pkt.addr_in1;
      i1.data_in<=pkt.data_in1;
      i1.write<=0;

      write_cycle;
    end
  endtask

  task write_cycle; start; addr; ack; data; ack; stop; endtask

  task start; i1.sda_in=1; endtask
  task addr; repeat(7) @(negedge i1.i2c_scl_out); endtask
  task data; repeat(8) @(negedge i1.i2c_scl_out); endtask
  task ack; i1.sda_in=0; @(negedge i1.i2c_scl_out); i1.sda_in=1; endtask
  task stop; i1.sda_in=1; endtask

endclass
```

---

# 🧪 Testbench (tb.sv)

```systemverilog
module tb;

packet pkt;
generator gen;
driver_count driv;   // Change driver here
mailbox m;

intf i2();

i2c_master dut(
  .addr_in(i2.addr_in),
  .data_in(i2.data_in),
  .write(i2.write),
  .clk_in(i2.clk_in),
  .reset_in(i2.reset_in),
  .sda_in(i2.sda_in),
  .i2c_sda_out(i2.i2c_sda_out),
  .i2c_scl_out(i2.i2c_scl_out),
  .read_out(i2.read_out)
);

always #5 i2.clk_in=~i2.clk_in;

initial begin
  i2.clk_in=0;
  i2.reset_in=1;
  #5 i2.reset_in=0;
end

initial begin
  m=new();
  gen=new(m);
  driv=new(m,i2);

  fork
    gen.run;
    driv.run;
  join
end

initial begin
  $dumpfile("waveform.vcd");
  $dumpvars;
  #1000 $finish;
end

endmodule
```

---
# ✅ **Advantages and Disadvantages**

---

## ✅ Advantages

### 1. Complete I2C Protocol Verification

* Verifies all major I2C operations including Start, Stop, Address, Read/Write, and Acknowledge.
* Ensures proper protocol compliance and reliable communication.

### 2. Class-Based Verification Environment

* Uses SystemVerilog classes such as packet, generator, and driver.
* Improves modularity, reusability, and maintainability.
* Makes the verification environment scalable.

### 3. Multiple Verification Modes

* Supports Count-Based, Randomized, Repeated Read, and Repeated Write testing.
* Helps validate the design under different operational scenarios.

### 4. Randomized Testing Support

* Random packet generation helps in identifying corner cases.
* Improves overall verification coverage and reliability.

### 5. FSM-Based RTL Design

* Finite State Machine simplifies control logic.
* Enhances code readability and debugging.

### 6. Waveform-Based Debugging

* GTKWave visualization helps analyze timing and signal behavior.
* Makes error detection and debugging easier.

### 7. Open-Source Tool Compatibility

* Uses Icarus Verilog and GTKWave.
* No dependency on expensive licensed tools.

### 8. Easy Customization

* Drivers and generators can be easily modified.
* New test scenarios can be added with minimal effort.

---

## ❌ Disadvantages

### 1. No UVM-Based Verification

* Does not use the industry-standard UVM methodology.
* Limited scalability for large verification projects.

### 2. Limited Functional Coverage

* No automated functional coverage collection.
* Verification completeness depends on manual testing.

### 3. No Scoreboard Implementation

* Does not include automatic result checking.
* Data correctness must be verified manually using waveforms.

### 4. Single Master-Slave Support

* Supports only basic single-master communication.
* Does not implement multi-master arbitration.

### 5. Limited Error Handling

* Advanced error scenarios such as bus contention and clock stretching are not covered.
* Some corner cases may remain untested.

### 6. No Performance Optimization

* Focus is on functional correctness rather than high-speed performance.
* Not optimized for high-frequency operation.

### 7. Manual Driver Selection

* Only one driver can be enabled at a time.
* Requires manual modification in the testbench.

### 8. Simulation-Only Verification

* Verified only in simulation.
* Not tested on FPGA or real hardware.

---

## 🚀 Future Enhancements

* Implement UVM-Based Verification Environment
* Add Functional and Code Coverage
* Develop Scoreboard for Automatic Checking
* Support Multi-Master I2C
* Add Error Injection Testing
* FPGA Implementation and Validation

---

## 📊 Waveforms

1. **Count-Based I2C Read/Write Waveform**
   <img width="1836" height="219" alt="count_based" src="https://github.com/user-attachments/assets/ac572ed3-afdc-4de2-bfd1-12939de197ff" />


2. **Randomized I2C Protocol Verification Waveform**
   <img width="1839" height="226" alt="randomized_i2c" src="https://github.com/user-attachments/assets/68bbaf8e-567c-4b8a-8432-b7388c604e89" />


3. **Repeated I2C Read Operation Waveform**
   <img width="1835" height="225" alt="repeated_read" src="https://github.com/user-attachments/assets/e1c5987c-db47-493d-a928-2461a75f97f2" />


4. **Repeated I2C Write-Read Operation Waveform**
   <img width="1837" height="222" alt="write_followed by read" src="https://github.com/user-attachments/assets/4b11b74a-e4f9-4db4-a18f-15e3fc50b4c4" />



# 👤 Author

**M Banu Shree**  
GitHub: https://github.com/banu0734  

---

