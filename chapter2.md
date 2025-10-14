# Chapter 2 – Relationship Between RTL, Hard Blocks, PHYs and Open Toolchains

## 2.1 FPGA Architecture: Soft Logic Versus Hard IP

Modern FPGAs are heterogenous systems combining **programmable logic** and **hardened blocks**.  The programmable logic (often called the *fabric*) consists of look‑up tables, flip flops and routing resources that can implement arbitrary RTL.  High‑speed interfaces such as PCI Express and USB 3.x typically use hardened blocks rather than purely fabric‑based designs because they require precise clock recovery, serializers/deserializers (SERDES), scramblers and training state machines operating at gigahertz rates.

To address this, vendors embed **hard blocks** for specific protocols.  For example, Xilinx 7‑Series FPGAs include a `PCIE_2_1` block that implements the PCIe Physical, Data Link and Transaction layers internally【319209819587374†L0-L5】, while Lattice ECP5 devices include SERDES macros and a hard PCS to support 8b/10b protocols.  These blocks present a high‑level interface to user logic – often an AXI‑stream bus – and hide the protocol complexity.  When using vendor tools and IP cores, designers instantiate these blocks and rely on them to implement the physical link negotiation, lane alignment, ACK/NAK generation and other features【852616939366164†L840-L856】.

### Soft Versus Hard Implementation

Open‑source developers can choose between using vendor hard IP (closed and vendor‑provided) and implementing the protocol in soft logic (fully accessible but involving more design work).  **Soft implementations** like *Yumewatari* implement 8b/10b encoding/decoding, symbol alignment and a basic Link Training and Status State Machine (LTSSM) in Verilog, operating over the FPGA’s general‑purpose SERDES【9†L5-L10】.  This provides visibility and modifiability but targets lower data rates (for example, PCIe Gen1 speeds) and involves calibration of the SERDES primitives.  **Hard IP** solutions such as the Xilinx `PCIE_2_1` block integrate the entire PHY and Data Link layer; open projects like `pcie_7x` simply wrap this block to expose its AXI‑stream interface【319209819587374†L0-L5】.  Because the underlying implementation is vendor‑supplied, the low‑level protocol cannot be modified by the user.

## 2.2 Internal and External PHYs

High‑speed serial protocols often separate the **Physical layer** into two components:

1. **The analog PHY** that implements the electrical drivers, equalization and clock recovery.  On FPGAs this is the SERDES macro and associated PLLs.
2. **The physical coding sublayer (PCS)** that performs 8b/10b or 128b/130b encoding/decoding, scramblers, lane deskew and link training sequences.

In many devices, these sub‑layers are implemented inside the chip (internal PHY) but some architectures allow an **external PHY** connected via the standardized **PIPE interface**.  For example, the Intel/Amaranth Daisho USB 3 core originally interfaced to an external PIPE PHY chip.  The `usb3_pipe` project explores using FPGA transceivers themselves as a PIPE PHY to **avoid external chips** and support multiple protocols (PCIe, SATA, DisplayPort)【22133856583036†L18-L40】.

### The PIPE Interface

The **PHY Interface for PCI Express (PIPE)** specification defines a uniform, packet‑oriented interface between the PCS and the Physical layer.  A PIPE PHY presents parallel data busses (`TX_DATA`, `RX_DATA`), control lines for start‑of‑packet, end‑of‑packet and polarity, and status signals for link training.  Higher layers see a synchronous 8b/10b encoded stream at the lane width (e.g., 8 bits per 125 MHz lane for Gen1/USB 3.0).  This separation allows the same PCS logic to work with different underlying SERDES technologies and even discrete PHY chips.  Projects such as `usb3_pipe` and `Yumewatari` implement a soft PCS that speaks PIPE to the FPGA transceivers, enabling the same logic to target multiple protocols.【22133856583036†L18-L40】

## 2.3 SERDES Documentation and Open Toolchains

Implementing a soft PHY or even just configuring the SERDES macros requires detailed knowledge of the bitstream.  The **Project Trellis** initiative documents the Lattice ECP5 architecture (including the SERDES) with the goal of enabling a free and open Verilog‑to‑bitstream toolchain【357389378787921†L31-L36】.  This documentation includes the configuration bits for the SERDES, PLL dividers, and 8b/10b gearbox, enabling projects like `ECP5‑PCIe` and *LUNA* to use the ECP5’s high‑speed transceivers entirely with open tools.

  The **F4PGA** project extends this idea to multiple vendors.  It provides architecture definitions for Xilinx 7‑Series, Lattice iCE40/ECP5 and QuickLogic devices and aims to offer an end‑to‑end open FPGA toolchain【143246778544798†L9-L12】.  F4PGA builds on repositories like Project X‑Ray (documenting Xilinx 7‑Series), Project IceStorm (iCE40) and Project Trellis (ECP5) to create a synthesis‑to‑bitstream flow using Yosys and nextpnr【143246778544798†L137-L165】.  While the logic fabric and some I/O blocks are well supported, documentation for high‑speed SERDES on many families is incomplete.  In Xilinx 7‑Series, the GTP/GTX transceivers have complex analog control registers and training sequences that are not yet fully reverse‑engineered.  Consequently, open designs often use vendor tools to configure the transceivers, even if the rest of the design is synthesized with open tools.

### Current State of SERDES Openness

The table below summarizes which FPGA families have sufficient SERDES documentation for open‑source usage.  ✔ indicates the family is supported by open bitstream documentation projects and can configure at least Gen1/USB 3.0 speeds; ✘ indicates that only proprietary tools can configure the transceivers.

| FPGA Family                    | Open SERDES Documentation? | Notes |
|--------------------------------|----------------------------|------|
| **Lattice ECP5**              | ✔ Full documentation (Project Trellis) | ECP5 SERDES supports up to 5 Gb/s; used by *LUNA* and `ECP5‑PCIe`【357389378787921†L31-L36】. |
| **Xilinx 7‑Series (Artix/Kintex)** | ⚠ Partial (Project X‑Ray) | Only basic configuration bits known; high‑speed tuning and equalization require Vivado.  Projects like `pcie_7x` wrap the built‑in PCIe block, avoiding custom SERDES configuration【319209819587374†L0-L5】. |
| **Xilinx UltraScale(+)/Versal** | ✘ No open documentation | GTY/GTYP transceivers and PCIe Gen4+ blocks are proprietary; open tools cannot configure them. |
| **Lattice Nexus (CrossLink‑NX/CertusPro‑NX)** | ⚠ Partial (Project Oxide) | Early reverse engineering shows promise but SERDES/PCIe bits are not yet fully published. |
| **Intel Cyclone/Arria**       | ✘ No open documentation | Transceivers and PCIe hard IP require Quartus. |
| **Microchip PolarFire**       | ✘ No open documentation | Current reverse engineering focuses on logic fabric; SERDES/PCIe bits unavailable. |

### Impact on Open‑Source PCIe/USB Development

Because only certain families offer open SERDES support, many open PCIe and USB projects target Lattice ECP5.  On Xilinx 7‑Series devices, developers typically rely on the vendor’s hard PCIe block (as in `pcie_7x`) or use vendor tools to configure the transceivers for protocols like USB 3 (`usb3_pipe`).  Due to limited SERDES documentation, fully open PHYs at Gen2/Gen3 speeds on mainstream platforms have not yet been demonstrated.

## 2.4 Toolchain Ecosystem and Design Flow

### Open Synthesis and Place‑and‑Route

Open FPGA toolchains typically follow a multi‑step flow: **synthesis** (Yosys), **technology mapping** (e.g., to LUT4 or LUT6 primitives), **place and route** (nextpnr) and **bitstream generation**.  Projects like F4PGA integrate these steps with architecture databases from X‑Ray, Trellis and IceStorm【143246778544798†L137-L165】.  When using open SERDES support (e.g. ECP5), designers can compile an entire high‑speed design from Verilog to bitstream without leaving the open flow.

### Hybrid Flows

For devices with incomplete SERDES support, designers often adopt a **hybrid flow**: use open tools for the logic fabric and vendor tools to insert hardened blocks or configure transceivers.  For example, the `usb3_pipe` project uses Vivado to configure Xilinx GTX/GTP transceivers but implements the 8b/10b PCS in open Migen code【22133856583036†L18-L40】.  The bitstream is thus partially opaque but still leverages open RTL for the majority of logic.  Similarly, some developers compile the entire design with Vivado but expose the hard IP via open wrappers (e.g., `pcie_7x` uses Vivado only as a back‑end and ensures no proprietary RTL is used【319209819587374†L0-L5】).  These approaches balance transparency with practicality until full SERDES documentation becomes available.

### Role of Protocol Analyzers and Simulation Tools

Debugging high‑speed links requires specialized tools.  **ngScopeClient** offers an open-source oscilloscope client capable of protocol decoding (PCIe and USB) and overlaying packet information on captured waveforms【166497749232038†L7-L16】.  In simulation, **warp‑pipe** models the PIPE interface, packetizing PCIe transactions into DLLPs and TLPs and transmitting them across distributed simulators【15367728790201†L23-L39】.  Such tools are essential companions to the open toolchain, allowing developers to validate custom PHY and protocol implementations without relying on proprietary analyzers.

## 2.5 Summary

This chapter has explored the **interplay between RTL, hardened IP, PHYs and open toolchains**.  We saw that while vendor‑supplied hard blocks greatly simplify implementing PCIe and USB3, they restrict transparency.  Open‑source projects respond with soft implementations (e.g., `Yumewatari`, `usb3_pipe`) and careful use of standardized interfaces like PIPE to decouple the PCS from SERDES.  Documentation efforts such as Project Trellis and F4PGA enable truly open flows on select devices, particularly the Lattice ECP5.  However, widespread adoption of fully open PCIe and USB3 cores hinges on further reverse‑engineering of SERDES architectures and standardization of interfaces across FPGA families.