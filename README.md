# pci-msi-starvation-issue

# The Core Problem:
   # The x86 Vector LimitIn the x86/x64 architecture, every CPU core has a Local APIC (Advanced Programmable Interrupt Controller)
   # An IDT (Interrupt Descriptor Table) on x86 has 256 hardware slots (vectors) per CPU core.
   # The Linux kernel reserves the first 32 vectors for CPU exceptions (like page faults) and another chunk for system interrupts (like timers or IPIs).
   # This leaves only about 200 remaining allocatable vectors per CPU core for all physical hardware combined (GPUs, NVMe arrays, NICs

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
                                            │     PLX PEX8725 Switch    │
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
3. The Failure Point ExplanationsSystem Baselines: Before the switchboards even kick off, Core 0's Local APIC has already consumed 1 vector for the onboard NIC, 4 vectors for the NVMe storage, and 32 vectors combined for the two primary motherboard FPGAs (1+4+32 = 37 vectors).Switchboard Saturation: As the system loops sequentially through the PEX8725 downstream bridges, each fully populated card consumes exactly 20 vectors (16 for the layout FPGA + 4 for the target devices).The Collision:Ports 06 through 0a (Switchboards 1 to 5) take up exactly 100 vectors (5 × 20), inflating the total allocated pool to 137 vectors.Moving onto bridge port 0b (Switchboard 6), the initialization logic safely registers the 6th sub-FPGA (+16) and instances 21, 22, and 23 (+3).This drives the raw hardware usage profile to 156 hardware vectors. Add the system's reserved core limits (~60 vectors for timers, scheduler, internal tasks), and Core 0's strict 256 hardware slot architecture limit is crossed.The Trigger Device: The failure explicitly occurs at address 0000:0b:00.4 (the 4th endpoint on Switchboard 6, which represents the 24th instance overall of device 0xABCD). When this block requests its MSI slice, pci_alloc_irq_vectors triggers -ENOSPC because Core 0 has run out of physical slots.
```
