# Chapter 5 – Toward a Fully Open PCIe Core: A Roadmap

The previous chapters highlight progress in open PCIe and USB3 development yet also reveal gaps, particularly in the Physical and Data Link layers.  Achieving a fully open PCIe core built with a free toolchain requires combining the successes of existing projects and addressing remaining challenges.  This chapter proposes a roadmap, focusing on leveraging open hardware platforms and being mindful of resource constraints.

## 5.1 Guiding Principles

1. **Leverage existing open‑source work** rather than reinventing the wheel.  Use proven soft PHY modules (e.g., `Yumewatari`/`ECP5‑PCIe` for PHY) and robust Transaction layer cores (e.g., `LitePCIe` or `Verilog‑PCIe`) while filling the gap in the Data Link layer.
2. **Target devices with open SERDES documentation**, notably the Lattice ECP5.  This avoids vendor licensing and allows the entire flow to remain open via Yosys, nextpnr and Trellis【357389378787921†L31-L36】【143246778544798†L137-L165】.
3. **Adopt modular architecture** with clearly defined interfaces: a soft PHY speaking PIPE; a DLL implementing credit‑based ACK/NAK; and a transaction layer providing AXI/Wishbone access and DMA.  Modularity enables community contributions and future protocol reuse.
4. **Utilize open analysis and simulation tools**.  Tools like warp‑pipe for TLP/DLLP simulation【15367728790201†L23-L39】 and ngScopeClient for physical signal analysis【166497749232038†L7-L16】 should accompany development to ensure correctness without proprietary analyzers.

## 5.2 Proposed Architecture

### 5.2.1 Physical Layer

- **Soft PHY on ECP5:** Adapt the 8b/10b PCS from `usb3_pipe` and `Yumewatari` to implement a generic PIPE PHY capable of both PCIe and USB3.  Use Trellis to configure the SERDES at 2.5 Gb/s and 5.0 Gb/s.  Validate signal integrity using open hardware boards (e.g., OrangeCrab, ULX3S) and ngScopeClient captures.
- **PIPE Abstraction:** Define a common PIPE interface module that can emit ordered sets (TS1/TS2 for PCIe, training sets for USB3) and accept training and status signals.  This will allow the same PHY to service multiple protocols and to be reused in SATA/DisplayPort projects.

### 5.2.2 Data Link Layer

- **Credit‑Based ACK/NAK Logic:** Implement a soft Data Link layer that replicates the PCIe credit mechanism.  The DLL must assign sequence numbers to TLPs, generate ACK/NAK DLLPs, maintain a replay buffer and handle flow control credits【852616939366164†L840-L856】.  The design can draw inspiration from the USB3 link layer, which already implements ACK/NRDY/ERDY handshakes and credit tokens【703161726344113†L103-L125】.  By studying LUNA’s link layer, developers can adapt techniques for reliable delivery to PCIe.
- **Link Initialization and Power Management:** Implement the state machine that transitions from Detect → Polling → Configuration → L0 and handles L0s/L1/L2 low‑power states.  Use warp‑pipe’s DLLP definitions to model link training and power management packets in simulation【15367728790201†L23-L39】.

### 5.2.3 Transaction Layer

- **Reuse LitePCIe/Verilog‑PCIe:** Once the DLL is in place, integrate an existing open Transaction layer core.  Both LitePCIe and Verilog‑PCIe offer robust TLP parsing, DMA engines and bus bridges【688402054328452†L307-L327】【895947797773709†L62-L129】.  Connecting them to a custom DLL requires aligning credit flows and may require minor wrapper modifications.
- **Configuration Space and Capability Registers:** Provide programmable Vendor ID, Device ID, BAR sizing and capability pointers (MSI/MSI‑X, PTM) so that Linux enumerates the device correctly.  Include multi‑vector MSI support as demonstrated in `pcie_7x`【13†L262-L270】.

### 5.2.4 Host Mode and Root Complex

- Although the initial goal is an endpoint, the roadmap should include a plan to develop an open Root Complex.  This involves implementing the upstream port’s transaction layer logic and host control registers.  The openPCIE project provides a starting point【182297874617818†L21-L29】.  For simple testing, the host mode can be run in simulation via warp‑pipe before hardware realization.

### 5.2.5 Software and Drivers

- **Linux Driver:** Build on existing LitePCIe drivers to support the new endpoint.  Provide uio or character device drivers to expose DMA and memory‑mapped registers.  Upstream contributions to the Linux kernel may eventually allow the open core to be recognized by generic drivers.
- **Zephyr Support:** For embedded systems, adapt Zephyr’s PCIe endpoint framework to use the open core.  This will enable open hardware devices to run lightweight RTOS tasks over PCIe.

## 5.3 Platform Selection Considerations

Implementing the proposed architecture requires hardware for development and testing.  Budget constraints guide the selection:

| Purpose | Recommended Platform | Estimated Cost (AUD) | Rationale |
|---|---|---|---|
| **Initial PHY/DLL development** | **Lattice ECP5 evaluation boards** (e.g., OrangeCrab or ULX3S) | 150–250 | These boards feature an ECP5‑45/85 with SERDES lanes wired to an FMC or PMOD and are supported by Trellis. |
| **PIPE testing** | **SFP breakout board + SFP‑to‑PCIe/USB adapters** | 50–100 | Allows connecting SERDES lanes to a host’s PCIe or USB port; used in `usb3_pipe` experiments. |
| **Transaction layer integration** | **Xilinx Artix‑7 boards** (e.g., Alinx AC7100) | 300–400 | Contains a `PCIE_2_1` block for comparison and can run `pcie_7x` for reference. |
| **Protocol analysis** | **Entry‑level oscilloscope with 1 GHz bandwidth** + **ngScopeClient** | 500–1,500 | Enables capturing training sequences and verifying electrical parameters. |

These choices aim to provide sufficient resources to test Gen1/Gen2 links while considering cost.  Communities like Fomu or ULX3S maintainers might loan boards or time on instruments, further lowering costs.

## 5.4 Development Phases

1. **Phase 1 – Soft PHY & PIPE Validation:** Adapt and merge `Yumewatari` and `usb3_pipe` into a unified soft PHY with a PIPE interface targeting ECP5.  Verify link training with ngScopeClient.  Deliver a module that can transmit and receive TS1/TS2 ordered sets and maintain 8b/10b encoding.
2. **Phase 2 – Data Link Layer Prototype:** Implement ACK/NAK logic, credit counters and replay buffer.  Use warp‑pipe to simulate and validate the DLL in co‑simulation.  Once stable, test on hardware by injecting error conditions and measuring recovery.
3. **Phase 3 – Transaction Layer Integration:** Connect the DLL to LitePCIe or Verilog‑PCIe’s TLP engine.  Implement configuration space with variable BAR sizes and MSI/MSI‑X.  Ensure enumeration on Linux and perform basic DMA transfers.
4. **Phase 4 – Performance Optimization and Compliance:** Tune SERDES settings for stable Gen2 (5 Gb/s) performance.  Validate electrical characteristics using ngScopeClient.  Optimize the replay buffer and credit scheduler for throughput.  Begin compliance testing against the PCI‑SIG specification.
5. **Phase 5 – Root Complex and Host Mode:** Adapt the DLL and TLP logic for an upstream port and incorporate openPCIE efforts to implement a root complex.  Demonstrate a self‑hosted system where both the host and device are implemented in open hardware.

## 5.5 Community and Collaboration

Achieving this roadmap requires **collaboration across projects and communities**.  Contributors from LUNA, LitePCIe, Verilog‑PCIe and openPCIE should coordinate to unify interfaces and share test infrastructure.  Funding from organisations like NLnet can sustain long‑term efforts, as seen with `pcie_7x` and LUNA.  A shared **specification for an open PCIe DLL** would encourage contributions and ensure compatibility with existing transaction layer cores.

## 5.6 Conclusion

Creating a fully open PCIe endpoint is within reach.  By building on the foundations laid by `ECP5‑PCIe`, `Yumewatari`, `usb3_pipe`, LitePCIe and Verilog‑PCIe, and using open documentation projects like Trellis and F4PGA, the community can assemble all layers of the PCIe stack within an open toolchain.  Combining a soft PIPE PHY with a new Data Link layer inspired by USB3 designs, and reusing existing Transaction layer engines, outlines a set of steps to complete the stack.  Selecting accessible platforms and open analysis tools makes the work feasible for small teams and hobbyists.  With sustained collaboration, fully open high‑speed interconnects may become more widely adopted.