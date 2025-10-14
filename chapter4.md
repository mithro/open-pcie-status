# Chapter 4 – Project Analyses: States, Successes and Challenges

This chapter offers a detailed analysis of key open‑source projects in the PCIe and USB 3 ecosystem.  For each project we discuss its goals, achievements, current status and limitations, focusing on how it fits within the goal of a fully open high‑speed interface.

## 4.1 ECP5‑PCIe

**Goal:** Provide a fully open PCIe endpoint implementation on the Lattice ECP5 FPGA.  The project combines a soft PHY (derived from *Yumewatari*) with open RTL for link training and seeks to implement the Data Link and Transaction layers in future.

**Successes:**

* Achieved link training on ECP5 evaluation boards at PCIe Gen1 and Gen2 speeds.  The core enters the L0 state and captures incoming Data Link Layer Packets (DLLPs) from the host, demonstrating correct TS1/TS2 exchange and lane synchronization【2†L315-L323】.
* Built entirely with open tools – synthesis using Yosys, place‑and‑route via nextpnr, and bitstream generation with Project Trellis【357389378787921†L31-L36】.  This proves that the ECP5 SERDES can be configured for PCIe purely via open bitstream documentation.
* Integrates easily with *LitePCIe* for the Transaction layer, as the project is designed to be a drop‑in PHY/DLL replacement for that core once DLL support is complete.

**Challenges and Limitations:**

* Currently **does not implement the Data Link layer**: the core does not generate ACK/NAK packets or maintain credit counters.  Without a proper DLL the host perceives no acknowledgements, preventing enumeration and meaningful data transfer.
* Signal integrity and timing closure at 5 Gb/s involve attention to placement and floorplanning.  Tuning of PLL and transmitter parameters has been used to improve Gen2 performance.
* Lacks a full root complex for host mode.  Although the openPCIE project attempts to open a root complex, combining the two is a long‑term goal.

## 4.2 Yumewatari

**Goal:** Demonstrate a fully soft PCIe Gen1 PHY on FPGAs.  The project was an early experiment by the *Amaranth* community to implement 8b/10b encoding, comma alignment and basic link training in Verilog or Migen.

**Successes:**

* Provided a **working 8b/10b encoder/decoder and LTSSM** that could establish a Gen1 link on Lattice ECP5 and Xilinx Artix‑7 hardware【9†L5-L10】.  This was one of the first public demonstrations of an open PCIe PHY running on an FPGA without vendor IP.
* Served as the foundation for later projects like **ECP5‑PCIe**.  Its code has been reused and extended to support Gen2 speeds and lane width negotiation.

**Challenges and Limitations:**

* **Limited feature set**: only supports single‑lane Gen1, no credit management or TLP handling.  It never evolved into a full Data Link layer or multi‑lane design.  As a result, the project is largely unmaintained and remains a prototype.
* Lacks formal specification compliance testing.  Without a standardized test plan, corner cases (e.g., error recovery) were never fully addressed.

## 4.3 LitePCIe

**Goal:** Provide a small‑footprint, configurable PCIe endpoint core with DMA and bus interfaces for FPGAs.  Part of the *LiteX* framework, LitePCIe aims to simplify integration of PCIe into custom SoCs.

**Successes:**

* Implements the **Transaction layer** including TLP packet parsing, reordering, scatter‑gather DMA, MSI/MSI‑X interrupts and a crossbar for multiple user interfaces【688402054328452†L307-L327】.  This allows developers to attach Wishbone or AXI devices to a PCIe BAR.
* Supports a wide range of FPGAs: **Xilinx 7‑Series, UltraScale(+) and Intel Cyclone V**.  For example, a LitePCIe endpoint on a Virtex UltraScale+ board has achieved over 30 Gb/s throughput on a Gen3 x8 link in an Antmicro demo.
* Provides a **Linux driver** and examples.  Projects using LitePCIe include video capture cards, SDR platforms and time synchronization cards; many of them are open hardware and rely on the DMA engine to stream data to/from the host.【688402054328452†L328-L344】.

**Challenges and Limitations:**

* Relies on the **vendor’s hardened PHY and DLL**.  A GitHub issue notes that LitePCIe uses the vendor’s TLP interfaces, meaning the Data Link layer and retry buffer are inside the closed IP【954422993286913†L245-L249】.  Consequently, the lower layers are implemented in vendor IP and not exposed.
* **Not portable to Lattice ECP5** (due to lack of hard PCIe block); implementing a soft PHY to connect to LitePCIe has not been demonstrated.
* The core supports a subset of PCIe features; peer‑to‑peer transactions and SR‑IOV are not included.

## 4.4 Verilog‑PCIe (Alex Forencich)

**Goal:** Provide a set of modular PCIe components – interfaces, bridges, arbiters and DMA engines – that can be used with vendor hard IP to build high‑performance PCIe devices across multiple FPGA families.

**Successes:**

* Offers a **generic internal TLP interface** that abstracts differences between Xilinx, Intel and other PCIe hard IPs.  Adapter modules like `pcie_us_if` (Xilinx UltraScale) or `pcie_s10_if` (Intel Stratix 10) translate between the vendor block and the generic interface【895947797773709†L43-L60】.
* Includes **bridges to AXI and AXI‑Lite**, as well as a **flexible DMA subsystem** that supports multi‑channel scatter‑gather transfers【895947797773709†L62-L129】.  These modules have been used in high‑bandwidth NICs (e.g., *Corundum*) and storage accelerators.
* Provides comprehensive **cocotb testbenches** through the `cocotbext‑pcie` project, enabling functional verification of endpoint designs in simulation.  The open test framework can model a PCIe root complex, assign BAR addresses and perform configuration accesses.

**Challenges and Limitations:**

* Requires the **vendor’s PCIe hard IP** for the Physical and Data Link layers; thus, it is not a full open solution.  However, because the interface to the hard IP is well abstracted, it is easier to swap in a future soft PHY.
* The design is oriented toward performance and flexibility and involves understanding PCIe protocol details.  It does not include a high‑level user interface like RIFFA.

## 4.5 openPCIE and PCIe Root Complex Efforts

**Goal:** Open the **root complex (host) side** of PCI Express to allow makers to build systems where both ends of the PCIe link are implemented in open hardware.  The project focuses on Xilinx Artix‑7 Gen2 root complexes【182297874617818†L10-L33】.

**Successes:**

* Addresses a missing component – while open endpoints exist, they rely on a proprietary root complex.  openPCIE aims to surround the closed Xilinx macro with open PIO logic and driver software【182297874617818†L21-L29】.
* Laid out a roadmap: design a mini PCIe backplane with standard connectors, implement open host hardware, develop layered driver software, and gradually phase out the proprietary macro over multiple projects【182297874617818†L49-L123】.  This systematic approach acknowledges the complexity of mixed‑signal SERDES and PLLs.

**Challenges and Limitations:**

* Still at an early stage.  The project currently wraps the Xilinx root complex but has not yet published a working open host design.
* No open alternative for the Xilinx Gen2 root macro; thus, the most interesting part (open host PHY) remains proprietary.

## 4.6 usb3_pipe

**Goal:** Use FPGA transceivers as a **USB 3.0 PIPE interface** to avoid external PHY chips and eventually support multiple protocols (PCIe, SATA, DisplayPort).  Provide an open alternative to commercial PIPE PHYs.

**Successes:**

* Demonstrated that a Xilinx Kintex‑7 FPGA can act as a USB 3 PHY when configured with a custom wrapper.  The core performs 8b/10b encode/decode, link training (including low frequency periodic signaling and ordered sets) and SKP alignment【22133856583036†L18-L40】.
* Allows connection to a host PC via an SFP‑to‑USB adapter, achieving link up and enumeration using a modified Daisho USB3 controller.  Simulation logs in the repository show the state transitions from Polling to U0 and the exchange of training sequences【22133856583036†L96-L111】.
* The design intentionally uses the **PIPE standard** so that the same transceiver wrapper could be repurposed for PCIe or SATA in the future【22133856583036†L18-L40】.

**Challenges and Limitations:**

* Currently limited to **Kintex‑7 and Artix‑7 boards**; configuration of the GTX transceiver requires Vivado.  F4PGA support is planned but not yet available【22133856583036†L42-L43】.
* Only provides the **PHY**; a separate USB3 controller core (such as Daisho or LUNA’s logic) is needed to implement the link and protocol layers.【22133856583036†L18-L40】
* Achieving compliance with the USB3 electrical specification involves tuning of equalization and jitter; the repository notes that the design is experimental and indicates that tuning may be necessary to meet the USB3 specification.

## 4.7 LUNA

**Goal:** Provide a complete **USB 2.0 and 3.x device** framework based on the open Amaranth (nMigen) HDL, including PHY, link and protocol layers.

**Successes:**

* Implements **low‑, full‑ and high‑speed USB (USB2)** using an off‑the‑shelf ULPI PHY and open gateware.  This portion is mature and used in GSG’s *Cynthion* USB test instrument.
* Provides experimental **SuperSpeed (USB 3.0) support** using two approaches: a **PIPE‑PHY** and a **SERDES‑based PHY**.  The current release supports SuperSpeed device mode via the PIPE interface; a SERDES‑based implementation for Lattice ECP5 is under active development【715347639244125†L47-L53】.
* Full **link and protocol layers** are implemented: LUNA handles link commands (NRDY/ERDY handshakes), transaction layer packet framing, endpoint management and class drivers【715347639244125†L154-L240】.  The project can enumerate as a USB 3.0 device and transfer data at SuperSpeed.
* Developed with open tools.  On ECP5 boards, the entire design (including SERDES) can be synthesized with Yosys/nextpnr thanks to Project Trellis【357389378787921†L31-L36】.

**Challenges and Limitations:**

* **Host mode** is not yet implemented【715347639244125†L141-L146】.  All current LUNA designs operate as USB devices; adding host capability requires implementing the host state machine, scheduling engine and root hub logic.
* **SuperSpeed support is labelled experimental**.  Achieving compliance on ECP5 involves meeting timing and signal integrity constraints.  Power‑management states (U1/U2) and link‑training edge cases are subjects of ongoing development.
* Board support is evolving.  For instance, the open hardware *Cynthion* board lacks the SuperSpeed physical lane; thus, testing is done on other ECP5 boards with custom connector modifications【715347639244125†L154-L240】.

## 4.8 pcie_7x

**Goal:** Provide a **fully open PCIe endpoint** on Xilinx 7‑Series devices that can be synthesized with openXC7 (F4PGA) tools.  Wraps the Xilinx `PCIE_2_1` hard block in open HDL and exposes simple AXI‑stream interfaces.

**Successes:**

* Achieved **enumeration and operation** of a PCIe Gen2 x1 endpoint on boards such as the Alinx AC7100 and *TimeCard* using fully open tools【319209819587374†L69-L80】.  The core responds to configuration TLPs, provides BAR registers and supports MSI interrupts.
* Enables experimentation with PCIe using open tools.  The openXC7 flow synthesizes the user logic and uses a pre‑configured wrapper for the hard block【319209819587374†L0-L5】.
* Added support for **multiple MSI vectors** and example Linux drivers.  Documentation provides step‑by‑step instructions for building and testing the endpoint using Docker【13†L262-L270】.

**Challenges and Limitations:**

* Limited to **Gen2 x1**.  The wrapper uses the Artix‑7/ Kintex‑7 `PCIE_2_1` block, which only supports up to Gen2 speeds.  Boards supporting x4 lanes are not yet targeted.
* **Relies entirely on the Xilinx hard macro** for the PHY, DLL and TLP engines.  Thus, while the wrapper is open, the core of the protocol remains closed.  There is no way to extend or modify the DLL or TLP layer behaviour.
* Only provides basic BAR interfaces and MSI; advanced features like DMA engines must be implemented externally or integrated from other projects (e.g., using LitePCIe or Verilog‑PCIe components).  Integration requires careful bus adaptation.

## 4.9 Warp‑Pipe

**Goal:** Provide a software library and protocol for connecting multiple hardware or simulation environments via a virtual PCIe/PIPE link.  Primarily used to connect the Renode simulator with QEMU or other models.

**Successes:**

* Implements packetization and depacketization of **TLPs and DLLPs**.  The library transmits them over a TCP socket using a custom protocol and recreates credit‑based flow control【15367728790201†L23-L39】.
* Supports **MSI and MSI‑X** routing and can model the hierarchy of switches and endpoints.  This allows complex PCIe topologies to be simulated in distributed systems.
* Used by Antmicro to test the Zephyr RTOS PCIe stack without physical hardware.  It also integrates with their `cocotbext‑pcie` environment to create co‑simulation between HDL and software.

**Challenges and Limitations:**

* Focuses solely on simulation; it cannot be used on hardware.  The physical layer and signal integrity are not modeled beyond the abstract PIPE protocol.
* Requires careful synchronization between simulators to avoid deadlock; proper credit management in software must match the hardware design being emulated.

## 4.10 ngScopeClient

**Goal:** Provide an **open‑source oscilloscope client** and protocol analyzer that can decode PCIe and USB waveforms and overlay protocol data on the captured signals.

**Successes:**

* Supports **complex protocol analysis**, combining waveform viewing with packet‑centric decoders for PCIe, Ethernet, USB and others【166497749232038†L7-L16】.  It can recover a PCIe 2.5 GT/s link training sequence, show the TS2 ordered sets and display collapsed TLPs and DLLPs for quick understanding.
* Works with multiple oscilloscope backends via libscopehal and can automate captures.  This empowers open hardware developers to debug high‑speed protocols without proprietary analyzers.
* Continues to evolve; its maintainers regularly add new decoders and features, such as 128b/130b support for PCIe Gen3/4.

**Challenges and Limitations:**

* Requires access to a suitable oscilloscope or digitizer; performance depends on the instrument’s bandwidth and memory depth.  Low‑cost scopes may not capture extended PCIe sessions.
* The tool does not generate signals; it only observes and decodes.  Thus, it must be used in conjunction with hardware under test, and cannot replace simulation frameworks like warp‑pipe.

## 4.11 RIFFA

**Goal:** Provide a **simple framework for FPGA–host communication over PCIe**.  RIFFA abstracts the PCIe protocol by exposing FIFO‑like interfaces on the FPGA and host side, enabling quick data transfer without requiring knowledge of TLPs.

**Successes:**

* Offers **portable cores** for Xilinx and Altera FPGAs (Gen1/Gen2) and host drivers for Linux and Windows【35†L275-L307】.  Developers can quickly exchange blocks of data between host and FPGA using straightforward send/receive APIs.
* Achieves high throughput via scatter‑gather DMA and multiple channels【35†L299-L307】, and is often used in research prototypes and university projects.

**Challenges and Limitations:**

* Relies on vendor PCIe IP for the PHY and DLL; thus it is not open at the lower layers.
* Provides little control over protocol details; for example, one cannot tune TLP size or implement custom capabilities.  The framework hides the PCIe transaction layer, so details such as TLP size or custom capabilities cannot be configured.

## 4.12 Summary

Overall, these project analyses show a variety of approaches.  Some projects provide open implementations of selected layers – for example, ECP5‑PCIe’s open SERDES configuration and LUNA’s USB device stack – while others rely on proprietary PHYs but offer high performance, as in LitePCIe and Verilog‑PCIe.  Many projects do not include open implementations of the Physical and Data Link layers.  Projects such as `pcie_7x` demonstrate that open toolchains can wrap vendor hard blocks; however, customizing or extending the protocol requires open PHYs and DLLs.  The next chapter proposes paths toward a fully open PCIe core based on these observations.