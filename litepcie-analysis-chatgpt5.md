# Chapter 6 – LitePCIe's PCIe Stack in Detail

LitePCIe positions itself as a "small footprint and configurable PCIe core" that covers the TLP layer, implements reordering, MSI/MSI-X and provides DMA and memory-mapped front-ends[[10]](sources.md#source10). This chapter examines the source code of LitePCIe and its vendor-specific PHY wrappers to document exactly which protocol layers are implemented inside LitePCIe and which functions depend on proprietary hard IP. It also contrasts the available PHY configurations, identifies the option that minimises reliance on vendor logic, and summarises the currently available information about resource usage. The architectural relationships described below trace directly to the modules and wrappers referenced in the cited source files.

## 6.1 LitePCIe's In-House Transaction Layer Processing

LitePCIe provides a full TLP pipeline in pure gateware. The packetizer inserts 3DW/4DW headers and merges header fields with payload words across widths up to 512 bits, demonstrating that the project owns the transaction-layer framing rather than delegating it to vendor IP[[14]](sources.md#source14). The depacketizer performs the inverse operation, extracting headers, realigning payloads and presenting request/completion streams to the rest of the design[[15]](sources.md#source15). On top of packetisation, the TLP controller manages request throttling, tag allocation, completion buffering and in-order delivery, all inside LitePCIe's Migen code[[16]](sources.md#source16).

The endpoint wrapper shows how these blocks surround any PHY: the PHY's streaming interface feeds directly into LitePCIe's depacketiser and the packetiser drives the PHY transmit port. The endpoint then exposes a crossbar that arbitrates between host-initiated traffic and FPGA-initiated traffic, again inside LitePCIe's logic[[17]](sources.md#source17).

```mermaid
flowchart LR
    subgraph LitePCIe Transaction Layer
        A[TLP Depacketiser]
        B[TLP Controller\n(tag + reorder)]
        C[TLP Packetiser]
        A --> B --> C
    end
    D[PHY Wrapper] --> A
    C --> D
    D -->|Streaming TLPs| VendorCore
```

*Figure 6.1: Transaction-layer modules enclosed by LitePCIe. The PHY wrapper only conveys raw TLP streams to the vendor hard IP.*

## 6.2 Vendor PHY Wrappers and Boundaries

### Xilinx UltraScale (PCIe Gen3 x8)

`USPCIEPHY` instantiates two independent datapaths for requester and completer traffic, performs clock-domain crossings/stride conversion and exposes MSI control registers before finally instantiating the proprietary `pcie_support` module from Xilinx. All flow-control, MSI, BAR sizing and link status signals are surfaced by LitePCIe, illustrating that the design retains responsibility for link management state while the hard IP is limited to the PHY/DLL interface[[18]](sources.md#source18). The wrapper explicitly generates the Tcl script that creates the `pcie3_ultrascale` IP and adds the vendor-provided Verilog shims (`s_axis_rq_adapt_*`, `m_axis_rc_adapt_*`, etc.), highlighting the boundary between LitePCIe and the hard block[[18]](sources.md#source18). The vendor support file shows the single instantiation of `pcie_us`, confirming the exact handoff point to Xilinx logic[[22]](sources.md#source22).

### Xilinx UltraScale+ (PCIe Gen3/Gen4 up to x16)

`USPPCIEPHY` extends the same structure to Gen4 speeds and permits up to 16 lanes with 512-bit user datapaths, but still channels all traffic through LitePCIe's CDC and width converters before invoking the vendor IP via the `pcie_support` instance[[19]](sources.md#source19). The assertions inside the class show that LitePCIe's own core supports 64/128/256/512-bit datapaths regardless of the narrower widths exposed by specific hard blocks[[19]](sources.md#source19), underscoring that the TLP layer is soft logic.

### Xilinx 7-Series (PCIe Gen1/Gen2 up to x8)

`S7PCIEPHY` connects the Xilinx `pcie_7x` block through AXI-Stream request/completion channels while LitePCIe handles PIPE clocking, MMCM generation, and resynchronising MSI and LTSSM status into the system domain[[20]](sources.md#source20). The wrapper also emits Tcl to create the IP, mirroring the UltraScale flow but showing that the vendor module only accepts already packetised TLP streams.

### Intel Cyclone V (PCIe Gen1/Gen2 up to x4)

`C5PCIEPHY` (not reproduced here for brevity) follows the same pattern, adapting between LitePCIe's native stream layout and Intel's Avalon-ST interface, with LitePCIe managing CDC, credit tracking signals and configuration space observation.

### Gowin GW5A (PCIe Gen1/Gen2 up to x4)

The Gowin wrapper interfaces with the vendor hard IP directly at the TLP layer: LitePCIe drives the `pcie_tl_tx_*` ports, receives `pcie_tl_rx_*` signals, and captures the vendor's credit counters into CSRs[[21]](sources.md#source21). All word swapping, byte-enable derivation and credit accounting logic is implemented in LitePCIe, so the vendor block provides only the PHY/DLL behaviour.

### Lattice L-SerDes (LFC-PNX)

Similarly, the Lattice wrapper converts between LitePCIe's endian-neutral layout and the vendor's TL bus, supplies credit initialisation flags and synchronises link-up indications, again leaving the transaction-layer sequencing and buffering entirely to LitePCIe[[21]](sources.md#source21).

## 6.3 Configuration Comparison

The table below summarises the PHY options currently in-tree and the scope of vendor functionality they require. In each case the LitePCIe core delivers the full TLP layer; the vendor contribution is restricted to the physical + data-link layers (and occasionally clock conditioning).

| PHY class | Supported widths | Lane counts | Vendor interface | Hard IP instantiation |
|-----------|-----------------|-------------|------------------|-----------------------|
| `USPCIEPHY` (Xilinx UltraScale) | 64/128/256 | x1/x2/x4/x8 | AXI-Stream requester/completer | `pcie3_ultrascale` via `pcie_support`[[18]](sources.md#source18)[[22]](sources.md#source22) |
| `USPPCIEPHY` (Xilinx UltraScale+) | 64/128/256/512 | x1/x2/x4/x8/x16 | AXI-Stream requester/completer | `pcie4_uscale_plus` / `pcie4c_uscale_plus` via `pcie_support`[[19]](sources.md#source19) |
| `S7PCIEPHY` (Xilinx 7-Series) | 64/128 | x1/x2/x4/x8 | AXI-Stream endpoint | `pcie_7x` hard block[[20]](sources.md#source20) |
| `C5PCIEPHY` (Intel Cyclone V) | 64 | x1/x2/x4 | Avalon-ST requester/completer | Altera PCIe hard IP (created via Qsys) |
| `GW5APCIEPHY` (Gowin GW5A) | 256 | x1/x4 | Native TL (sop/eop/data/be + credits) | `PCIE_Controller_Top` core[[21]](sources.md#source21) |
| `LFCPNXPCIEPHY` (Lattice L-SerDes) | 128 | x4 | Native TL (valid/ready/data/be + credits) | Lattice PCIe controller with LMMI config[[21]](sources.md#source21) |

The Gowin configuration demands the least vendor functionality: the vendor IP exposes only TL-level handshakes and credits, leaving LitePCIe to implement MSI handling, byte-lane steering and credit accounting. The Lattice wrapper offers a similar boundary. In contrast, the Xilinx wrappers rely on vendor adaptation shims but still feed/consume fully formed TLPs, keeping LitePCIe in charge of packet construction, tags and flow control.

## 6.4 Capabilities Beyond Vendor Hard Blocks

LitePCIe's packetiser/depacketiser support 512-bit datapaths, even though the Xilinx UltraScale Gen3 hard block tops out at 256 bits. The UltraScale+ wrapper demonstrates how LitePCIe scales its own internal datapath to 512 bits while adapting to the widths supported by each vendor IP[[14]](sources.md#source14)[[19]](sources.md#source19). The Gowin and Lattice ports further show features (credit counters, byte-swapping) that would normally be buried inside proprietary TLP controllers. LitePCIe also integrates PTM packet support and completion reordering in software-defined logic[[16]](sources.md#source16)[[17]](sources.md#source17), features that are typically fixed-function in vendor endpoints.

## 6.5 Resource Usage Information

LitePCIe emphasises a "small footprint" in its own documentation but does not publish a vendor-versus-LitePCIe utilisation table in the repository or accompanying documentation[[10]](sources.md#source10). The source tree contains no utilisation reports for the reference designs inspected here. Consequently, while the architectural evidence clearly shows that LitePCIe implements the transaction layer and ancillary features itself, no public quantitative comparison with vendor IP footprints is currently available.

## 6.6 Key Takeaways

1. LitePCIe implements header formatting, tag management, completion reordering and crossbar routing in portable HDL, independent of vendor hard IP.
2. Vendor wrappers supply only the PHY/DLL boundary and clocking, with LitePCIe feeding them TLP streams.
3. Gowin and Lattice configurations expose the narrowest vendor interface (pure TL handshake), maximising LitePCIe's control of the stack.
4. LitePCIe supports datapaths and features (e.g. 512-bit user widths, PTM, DMA crossbar) that require a soft TLP layer even when the vendor hard IP lacks equivalent capabilities.
5. Public resource utilisation numbers are not presently available, so further measurement would be needed for direct comparisons.
