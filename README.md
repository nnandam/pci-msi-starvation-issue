# pci-msi-starvation-issue
# System setup
```text
 Broadwell 6 core multi-root complex SOC
 PCIE SWITCH
 Two Onboard FPGA
 NVME
 Ethernet
 8 PCIE Switch Boards connected to Downports of Mother Board PCIE Switch
 Each PCIE Switch Board has 1 FPGA
 Each PCI Switch board has 4 instances of PCIE Device VendorID 0x1234 and DeviceID 0xABCD
#
```
# 1. PCIe Physical Topology Block Diagram
```text
                 ┌────────────────────────────────────────────────────────┐
                 │                  Broadwell 6-Core SoC                  │
                 │   [Core 0] [Core 1] [Core 2] [Core 3] [Core 4] [Core 5]│
                 │                    (Local APICs)                       │
                 └───────────────────────────┬────────────────────────────┘
                                             │
                              ┌──────────────┴──────────────┐
                              │ PCIe Root Complex / Root Port│
                              └──────────────┬──────────────┘
                                             │
      ┌────────────────────────┬─────────────┴────────────┬────────────────────────┐
      │                        │                          │                        │
┌─────┴──────┐           ┌─────┴──────┐            ┌──────┴───────┐         ┌──────┴───────┐
│Onboard NIC │           │Onboard NVMe│            │Onboard FPGA 1│         │Onboard FPGA 2│
│(10GbE / 1v)│           │(Gen3 x4/4v)│            │ (16 vectors) │         │ (16 vectors) │
└────────────┘           └────────────┘            └──────────────┘         └──────────────┘
                                                          │
                                            ┌─────────────┴─────────────┐
                                            │     PCIE        Switch    │
                                            └─────────────┬─────────────┘
                                                          │
          ┌───────────────┬───────────────┬───────────────┼───────────────┬───────────────┬───────────────┬───────────────┐
          │               │               │               │               │               │               │               │
     ┌────┴────┐     ┌────┴────┐     ┌────┴────┐     ┌────┴────┐     ┌────┴────┐     ┌────┴────┐     ┌────┴────┐     ┌────┴────┐
     │Slot 1   │     │Slot 2   │     │Slot 3   │     │Slot 4   │     │Slot 5   │     │Slot 6   │     │Slot 7   │     │Slot 8   │
     │Switchbrd│     │Switchbrd│     │Switchbrd│     │Switchbrd│     │Switchbrd│     │Switchbrd│     │Switchbrd│     │Switchbrd│
     └────┬────┘     └────┬────┘     └────┬────┘     └────┬────┘     └────┬────┘     └────┬────┘     └────┬────┘     └────┬────┘
          │               │               │               │               │               │               │               │
     ┌────┼────┐     ┌────┼────┐     ┌────┼────┐     ┌────┼────┐     ┌────┼────┐     ├────┼────┐     ┌────┼────┐     ┌────┼────┐
     │FPGA│Dev1│     │FPGA│Dev5│     │FPGA│Dev9│     │FPGA│Dev13│    │FPGA│Dev17│    │FPGA│Dev21│    │FPGA│Dev25│    │FPGA│Dev29│
     │(16)│(1) │     │(16)│(1) │     │(16)│(1) │     │(16)│(1)  │    │(16)│(1)  │    │(16)│(1)  │    │(16)│(1)  │    │(16)│(1)  │
     │    │Dev2│     │    │Dev6│     │    │Dev10│    │    │Dev14│    │    │Dev14│    │    │Dev22│    │    │Dev26│    │    │Dev30│
     │    │(1) │     │    │(1) │     │    │(1)  │    │    │(1)  │    │    │(1)  │    │    │(1)  │    │    │(1)  │    │    │(1)  │
     │    │Dev3│     │    │Dev7│     │    │Dev11│    │    │Dev15│    │    │Dev15│    │    │Dev23│    │    │Dev27│    │    │Dev31│
     │    │(1) │     │    │(1) │     │    │(1)  │    │    │(1)  │    │    │(1)  │    │    │(1)  │    │    │(1)  │    │    │(1)  │
     │    │Dev4│     │    │Dev8│     │    │Dev12│    │    │Dev16│    │    │Dev16│    │    │💥Dev24│   │    │Dev28│    │    │Dev32│
     │    │(1) │     │    │(1) │     │    │(1)  │    │    │(1)  │    │    │(1)  │    │    │(FAIL)│    │    │(1)  │    │    │(1)  │
     └────┴────┘     └────┴────┘     └────┴────┘     └────┴────┘     └────┴────┘     └────┴────┘     └────┴────┘     └────┴────┘
 ```
 
# lspci -tv Bus Enumeration OutputHere is how the kernel sees the device hierarchy on the bus tree during initialization.
# The numbers in parentheses show the running count of assigned MSI vectors focusing on Core 0's vector exhaustion barrier:
```text
-[0000:00]-+-00.0  Intel Corporation Broadwell Host Bridge
           +-01.0-[01]--00.0  Intel Corporation Ethernet Connection (1 vector)
           +-01.1-[02]--00.0  Non-Volatile Memory System NVMe SSD (4 vectors)
           +-02.0-[03]--00.0  Xilinx FPGA Board Base 1 (16 vectors)
           +-02.1-[04]--00.0  Xilinx FPGA Board Base 2 (16 vectors)
           \-03.0-[05-0f]--00.0-[06-0f]--+-01.0-[06]--+-00.0  FPGA Switchboard 1 (16 vectors)
                                         |            +-00.1  Device [1234:abcd] (1 vector)
                                         |            +-00.2  Device [1234:abcd] (1 vector)
                                         |            +-00.3  Device [1234:abcd] (1 vector)
                                         |            \-00.4  Device [1234:abcd] (1 vector)
                                         +-02.0-[07]--+-00.0  FPGA Switchboard 2 (16 vectors)
                                         |            +-00.1  Device [1234:abcd] (1 vector)
                                         |            ... [Devices 6, 7, 8 (1 vector each)]
                                         +-03.0-[08]--+-00.0  FPGA Switchboard 3 (16 vectors)
                                         |            ... [Devices 9, 10, 11, 12 (1 vector each)]
                                         +-04.0-[09]--+-00.0  FPGA Switchboard 4 (16 vectors)
                                         |            ... [Devices 13, 14, 15, 16 (1 vector each)]
                                         +-05.0-[0a]--+-00.0  FPGA Switchboard 5 (16 vectors)
                                         |            +-00.1  Device [1234:abcd] (1 vector)
                                         |            +-00.2  Device [1234:abcd] (1 vector)
                                         |            +-00.3  Device [1234:abcd] (1 vector)
                                         |            \-00.4  Device [1234:abcd] (1 vector) -> (Total vectors allocated: 151)
                                         \-06.0-[0b]--+-00.0  FPGA Switchboard 6 (16 vectors) -> (Allocated: 167)
                                                      +-00.1  Device [1234:abcd] (1 vector)  -> (Allocated: 168)
                                                      +-00.2  Device [1234:abcd] (1 vector)  -> (Allocated: 169)
                                                      +-00.3  Device [1234:abcd] (1 vector)  -> (Allocated: 170)
                                                      \-00.4  💥 Device [1234:abcd] (FAIL: -ENOSPC)
```

```text
3. The Failure Point ExplanationsSystem Baselines: Before the switchboards even kick off,
 Core 0's Local APIC has already consumed 1 vector for the onboard NIC,
 4 vectors for the NVMe storage,
 and 32 vectors combined for the two primary motherboard FPGAs (1+4+32 = 37 vectors).
 Switchboard Saturation: As the system loops sequentially through the PEX8725 downstream bridges,
  each fully populated card consumes exactly 20 vectors (16 for the layout FPGA + 4 for the target devices).
  The Collision:Ports 06 through 0a (Switchboards 1 to 5) take up exactly 100 vectors (5 × 20),
  inflating the total allocated pool to 137 vectors.
  Moving onto bridge port 0b (Switchboard 6), the initialization logic safely registers the 6th sub-FPGA (+16) and
  instances 21, 22, and 23 (+3).This drives the raw hardware usage profile to 156 hardware vectors.
  Add the system's reserved core limits (~60 vectors for timers, scheduler, internal tasks),
  and Core 0's strict 256 hardware slot architecture limit is crossed.
  # The Trigger Device: The failure explicitly occurs at address 0000:0b:00.4 (the 4th endpoint on Switchboard 6,
  # which represents the 24th instance overall of device 0xABCD). # When this block requests its MSI slice,
 #  pci_alloc_irq_vectors triggers -ENOSPC because Core 0 has run out of physical slots.
```


# SOLUTION
# a two-pronged solution: optimize the hardware request footprint (reducing the FPGAs from 16 vectors to 1 vector)
# and configure the host kernel/architecture to balance interrupts across all 6 Broadwell cores using Interrupt Remapping.

Fix 1: Optimizing the Code to Request 1 MSI Vector per FPGABy changing the driver allocation loop or 
updating the FPGA endpoint configurations to only request 1 vector instead of 16,
you instantly wipe out the vector footprint.
# The Math After Optimization:
# 2 On-board FPGAs = 2 vectors (Previously 32)
# 8 Switchboard FPGAs = 8 vectors (Previously 128)
# 32 Endpoint Devices = 32 vectors 
# Onboard NIC + NVMe = 5 vectors 
# At 47 vectors total, the entire system easily fits well under Core 0's native ~200 free vector limit. 
# Furthermore, because they are requesting single vectors, you completely eliminate the multi-MSI contiguous allocation rule, eradicating x86 IDT table fragmentation.


# 🛑 1. BEFORE (Starvation and Saturated Core 0)The Problem: 
# Every single FPGA requests 16 vectors, forcing consecutive allocations that jam the x86 IDT table.
# The Result: All interrupts are statically pinned to CPU0. When the 24th instance tries to initialize, it hits vector exhaustion (-ENOSPC), failing to ever appear in /proc/interrupts.
```text
           CPU0       CPU1       CPU2       CPU3       CPU4       CPU5
  0:         42          0          0          0          0          0  IR-IO-APIC    2-edge      timer
  1:         10          0          0          0          0          0  IR-IO-APIC    1-edge      i8042
  8:          1          0          0          0          0          0  IR-IO-APIC    8-edge      rtc0
 40:     102450          0          0          0          0          0   PCI-MSI      0000:01:00.0  eth0
 41:      50124          0          0          0          0          0   PCI-MSI      0000:01:01.0  nvme0q0
 42:       4810          0          0          0          0          0   PCI-MSI      0000:01:01.0  nvme0q1
...
 45:          0          0          0          0          0          0   PCI-MSI      0000:02:00.0  fpga_base1_0
... [Vectors 46 to 60 are eaten up sequentially by the rest of fpga_base1's 16 vectors] ...
 61:          0          0          0          0          0          0   PCI-MSI      0000:02:01.0  fpga_base2_0
... [Vectors 62 to 76 are eaten up sequentially by the rest of fpga_base2's 16 vectors] ...
 77:       1200          0          0          0          0          0   PCI-MSI      0000:06:00.0  fpga_swbd1_0
... [Vectors 78 to 92 are eaten up sequentially by fpga_swbd1's 16 vectors] ...
 93:        844          0          0          0          0          0   PCI-MSI      0000:06:00.1  dev1
... [Switchboards 2, 3, 4, and 5 keep demanding 16 vectors + 4 dev vectors each, cascading rapidly] ...
210:          5          0          0          0          0          0   PCI-MSI      0000:0a:00.4  dev20
211:          0          0          0          0          0          0   PCI-MSI      0000:0b:00.0  fpga_swbd6_0
... [Vectors 212 to 226 are eaten up sequentially by fpga_swbd6's 16 vectors] ...
227:        102          0          0          0          0          0   PCI-MSI      0000:0b:00.1  dev21
228:         94          0          0          0          0          0   PCI-MSI      0000:0b:00.2  dev22
229:         12          0          0          0          0          0   PCI-MSI      0000:0b:00.3  dev23
ERR:          0

```
# 2. AFTER (Optimized Vectors + Interrupt Remapping Active)
# The Fix: Every FPGA footprint is optimized down to 1 vector instead of 16.
# The Result: The kernel loads intel_iommu=on intremap=on and maps the interrupts using the IR-PCI-MSI framework.
# It leverages PCI_IRQ_AFFINITY and irqbalance to distribute vectors evenly across all 6 cores,
# allowing dev24 (and all subsequent cards) to register flawlessly.

```text
           CPU0       CPU1       CPU2       CPU3       CPU4       CPU5
  0:         42          0          0          0          0          0  IR-IO-APIC    2-edge      timer
  1:         10          0          0          0          0          0  IR-IO-APIC    1-edge      i8042
 40:     102450          0          0          0          0          0  IR-PCI-MSI   10240-edge   eth0
 41:          0      50124          0          0          0          0  IR-PCI-MSI   20480-edge   nvme0q0
 42:          0          0       4810          0          0          0  IR-PCI-MSI   20481-edge   nvme0q1
 43:          0          0          0       2100          0          0  IR-PCI-MSI   32768-edge   fpga_base1
 44:          0          0          0          0       1420          0  IR-PCI-MSI   34816-edge   fpga_base2
 45:          0          0          0          0          0       3110  IR-PCI-MSI   40960-edge   fpga_swbd1
 46:       1500          0          0          0          0          0  IR-PCI-MSI   40961-edge   dev1
 47:          0       1844          0          0          0          0  IR-PCI-MSI   40962-edge   dev2
 48:          0          0       1120          0          0          0  IR-PCI-MSI   40963-edge   dev3
 49:          0          0          0       1992          0          0  IR-PCI-MSI   40964-edge   dev4
...
 74:          0          0          0          0          0        482  IR-PCI-MSI   49152-edge   fpga_swbd6
 75:        312          0          0          0          0          0  IR-PCI-MSI   49153-edge   dev21
 76:          0        288          0          0          0          0  IR-PCI-MSI   49154-edge   dev22
 77:          0          0        194          0          0          0  IR-PCI-MSI   49155-edge   dev23
 78:          0          0          0        512          0          0  IR-PCI-MSI   49156-edge   dev24  <-- SUCCESS!
 79:          0          0          0          0        411          0  IR-PCI-MSI   49157-edge   dev25
ERR:          0

```
