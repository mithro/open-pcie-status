# Verilog-PCIe Architecture Analysis (ChatGPT-5)

Verilog-PCIe is a collection of PCI Express endpoint building blocks that keeps transaction-layer processing in portable HDL while relying on vendor IP only for the physical/link interface. The project ships parameterisable DMA engines, AXI bridges and per-vendor shims that adapt raw PCIe hard blocks into a generic TLP stream bus.[^repo-readme] This chapter follows the same outline as the LitePCIe report but tracks the exact code boundaries inside Alex Forencich's repository. Every link below points to the precise file and line range on the `25156a9a162c41c60f11f41590c7d006d015ae5a` commit of https://github.com/alexforencich/verilog-pcie.

[^repo-readme]: [README.md](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/README.md#L11-L83)

## 7.1 Generic TLP Pipeline Under a Vendor-Neutral Interface

The core PCIe datapath inside Verilog-PCIe is built around segmented TLP streams that are independent of any specific FPGA. The UltraScale adapter (`pcie_us_if`) exposes four AXI-Stream channels (RC, RQ, CQ, CC) from the hard block and converts them into internal request/completion TLP streams used by the DMA subsystem.[^pcie-us-if-ports] Each channel delegates to specialised submodules (`pcie_us_if_rc`, `pcie_us_if_rq`, `pcie_us_if_cq`, `pcie_us_if_cc`) that align headers, translate per-beat metadata and forward segmented payloads.[^pcie-us-if-rc][^pcie-us-if-rq]

`pcie_us_if_rq` demonstrates that request TLP generation (header formatting, sequence tracking, flow-control accounting) runs in open-source logic: it ingests read/write request headers and payload data from the DMA engine and applies vendor credits (`cfg_fc_*`) to throttle issuance before presenting AXI-Stream beats to the Xilinx block.[^pcie-us-if-rq-body] The corresponding completion path (`pcie_us_if_rc`) strips user bits from the vendor stream, reconstructs TLP headers and signals errors back into the DMA fabric, again without vendor involvement.[^pcie-us-if-rc-body]

Outside UltraScale parts, the Intel Stratix 10 (`pcie_s10_if`) and P-Tile (`pcie_ptile_if`) shims provide the same translation between vendor-specific AVST interfaces and the internal segmented TLP bus. They map native flow-control and function-number signals into the generic request/completion channels while emitting sequence numbers for DMA read/write tracking.[^pcie-s10-if][^pcie-ptile-if] This uniform boundary allows the rest of the repository (DMA engines, BAR decoders, AXI bridges) to operate identically across vendors.

## 7.2 DMA and AXI Bridges Implement the Transaction Layer

The `pcie_us_axi_dma` top-level module instantiates independent read and write pipelines (`pcie_us_axi_dma_rd`, `pcie_us_axi_dma_wr`) that translate descriptor queues into PCIe request TLPs and marshals completions back into AXI bursts.[^pcie-us-axi-dma] Each pipeline issues PCIe read/write commands from software-provided descriptors, arbitrates tags, tracks in-flight operations and handles retransmission via sequence numbers—functions typically provided by proprietary transaction layers.[^pcie-us-axi-dma-rd] Flow control is maintained by explicitly consuming the vendor credit counters exported by `pcie_us_if` (`pcie_tx_fc_*` inputs), reinforcing that TLP scheduling occurs in Verilog.[^pcie-us-axi-dma-fc]

BAR access is handled by AXI master bridges. The `pcie_axi_master` wrapper instantiates dedicated reader/writer modules that transform PCIe completions and requests into AXI transactions, including burst assembly, outstanding request tracking and completion error handling.[^pcie-axi-master] For simpler register blocks, `pcie_axil_master_minimal` limits itself to aligned 32-bit accesses and generates completer aborts for unsupported operations, a behaviour directly coded in Verilog rather than offloaded to vendor IP.[^pcie-axil-master-minimal]

Auxiliary primitives (`pcie_tlp_fifo`, `pcie_tlp_demux_bar`, `dma_if_pcie_us`, etc.) form the plumbing that distributes TLPs between DMA clients and BAR endpoints. These modules realign payloads, mux multiple requesters, and enforce ordering using segmented dual-port RAMs; they never call into vendor libraries.[^pcie-tlp-fifo][^dma-if-pcie-us]

## 7.3 Vendor Adapter Boundaries

Example designs show the explicit seam between Verilog-PCIe logic and proprietary cores. On Xilinx UltraScale+ boards, the FPGA top-level instantiates the Xilinx generated IP (e.g. `pcie4_uscale_plus`) and wires its AXI-Stream ports directly into `example_core_pcie_us`, which in turn connects to `pcie_us_if` and the generic DMA/AXI subsystems.[^vcu118-top][^example-core-pcie-us] The vendor block exports user clock/reset, link status, and AXI-Stream buses; all packet construction, BAR mapping and DMA scheduling sit on the Verilog side of the boundary.

Intel designs follow the same pattern. `example_core_pcie_ptile` connects the P-Tile AVST signals to `pcie_ptile_if`, which feeds the shared DMA fabric while capturing vendor credit counters and bar/function routing bits.[^example-core-ptile] Stratix 10 designs reuse `pcie_s10_if`, translating H-Tile/L-Tile interfaces before injecting requests into the common TLP muxes.[^example-core-s10]

The adapters also manage MSI/MSI-X signalling without vendor assistance. `pcie_us_msi` and `pcie_msix` create write-request TLPs directly, and the Xilinx adapter surfaces MSI configuration fields via AXI-Lite registers (`cfg_interrupt_msix_*` outputs) that the Verilog code programs.[^pcie-us-msi][^pcie-msix]

## 7.4 Configuration and Capability Extraction

Verilog-PCIe reads link capabilities from vendor status ports but interprets them itself. `pcie_us_if` connects the Xilinx configuration interface, capturing maximum payload/read sizes and extended tags into internal registers that drive DMA limits and descriptor acceptance.[^pcie-us-if-config] The module multiplexes management reads/writes onto the vendor `cfg_mgmt` bus while exposing only the necessary status bits to the rest of the core.[^pcie-us-if-mgmt]

For Intel P-Tile endpoints, `pcie_ptile_cfg` provides a dedicated configuration shim that translates the P-Tile CSR layout into the same generic signals consumed by the DMA engines (BAR apertures, function numbers, interrupt enables).[^pcie-ptile-cfg] Stratix 10 designs use the AVST sideband signals for BAR range selection, which `pcie_s10_if` converts into BAR IDs for the generic request router.[^pcie-s10-if-bar]

### Functionality Comparison Table

| Functionality | Vendor Hard IP | Verilog-PCIe | Notes |
|---|---|---|---|
| **SERDES/PHY + clocking** | ✓ | ✗ | Provided by `pcie4_uscale_plus`/Intel HIP blocks instantiated in the examples.[^vcu118-top][^example-core-ptile] |
| **LTSSM and DLLP handling** | ✓ | ✗ | Managed inside the vendor core that feeds AXI-Stream/AVST buses.[^pcie-us-if][^pcie-s10-if] |
| **PHY-level flow-control credits** | ✓ | ✗ | Vendors expose credit counters (`cfg_fc_*`, `hip_tx_sched_*`) consumed by the adapters.[^pcie-us-if-rq][^pcie-ptile-if] |
| **Standard configuration space (0x00–0xFF)** | ✓ | ✗ | Hard IP retains the base configuration registers and surfaces them via `cfg_mgmt`/CSR ports.[^pcie-us-if-config][^pcie-ptile-cfg] |
| **Extended configuration space access** | ✗ | ✓ | `pcie_us_if` multiplexes management transactions and captures extended capabilities in soft logic.[^pcie-us-if-mgmt] |
| **TLP packet construction** | ✗ | ✓ | `pcie_us_if_rq` formats request headers, sequences tags and enforces credits.[^pcie-us-if-rq-body] |
| **TLP packet parsing** | ✗ | ✓ | `pcie_us_if_rc` reconstructs completion headers and status flags.[^pcie-us-if-rc-body] |
| **BAR address decoding & AXI bridging** | ✗ | ✓ | `pcie_axi_master` translates TLPs into AXI transactions with range checks.[^pcie-axi-master] |
| **Scatter/gather DMA engines** | ✗ | ✓ | `pcie_us_axi_dma` and its read/write pipelines implement descriptor-driven DMA.[^pcie-us-axi-dma][^pcie-us-axi-dma-rd] |
| **MSI/MSI-X generation** | ✗ | ✓ | `pcie_us_msi` and `pcie_msix` craft interrupt write TLPs and manage tables.[^pcie-us-msi][^pcie-msix] |
| **Request/completion routing & multiplexing** | ✗ | ✓ | `dma_if_pcie_us` arbitrates clients over segmented TLP streams.[^dma-if-pcie-us] |
| **TLP-level flow control** | ✗ | ✓ | DMA engines consume exposed credits to throttle outstanding transactions.[^pcie-us-axi-dma-fc] |

## 7.5 Verification Infrastructure

The repository includes comprehensive cocotb and MyHDL testbenches that validate the soft transaction layer without any vendor simulation models. `test_pcie_us.py` constructs a behavioural model of the UltraScale+ hard block interface, exercises configuration, request/completion, and flow-control paths, and asserts correct TLP decoding in the `pcie_us_if` logic.[^test-pcie-us] Additional tests cover the DMA pipelines across widths (64/128/256/512 bits), ensuring descriptor handling, AXI burst generation and completion error reporting operate solely under Verilog control.[^test-pcie-us-axi-dma][^test-dma-if-pcie-us]

CI automation runs these tests via GitHub Actions (`Regression Tests` workflow), providing regression coverage across the interface adapters and DMA subsystems independently of vendor tooling.[^ci-workflow]

## 7.6 Vendor IP Dependency Matrix

The following table summarises each adapter, its supported data widths/lane counts, and the specific vendor IP boundary it touches. In every case, Verilog-PCIe owns the TLP formatting, DMA scheduling and AXI translation; proprietary IP delivers only the SERDES/PHY/DLL abstraction plus clocking.

| Adapter module | Target families | User datapath width | Vendor IP instance | Interface boundary |
|----------------|-----------------|---------------------|--------------------|--------------------|
| `pcie_us_if` | Xilinx UltraScale/UltraScale+ | 64/128/256/512-bit AXI-Stream (RQ/RC/CQ/CC) | `pcie3_ultrascale`, `pcie4_uscale_plus`, `pcie4c_uscale_plus` | AXI-Stream packet interface, configuration/status sidebands, MSI control[^pcie-us-if][^vcu118-top] |
| `pcie_s10_if` | Intel Stratix 10 H-/L-Tile | 1×256-bit AVST segments | Stratix 10 Hard IP for PCIe | AVST RX/TX segments, flow control credits, MSI signals[^pcie-s10-if][^example-core-s10] |
| `pcie_ptile_if` | Intel Stratix 10 DX / Agilex P-Tile | up to 4×128-bit AVST segments | Intel P-Tile PCIe HIP | AVST RX/TX segments with prefix headers, credit limits, function routing[^pcie-ptile-if][^example-core-ptile] |

The Xilinx examples further highlight that the vendor netlists are created by Vivado IP integrator and handed raw AXI-Stream data; all request/completion semantics are managed in open Verilog modules (`example_core_pcie_us`, `dma_if_pcie_us`, etc.).[^example-core-pcie-us]

## 7.7 Key Observations

1. Transaction-layer packetisation, tag bookkeeping, flow-control and DMA scheduling are implemented entirely in Verilog modules such as `pcie_us_if_*`, `pcie_us_axi_dma_*` and `pcie_axi_master_*`; vendor IP remains a PHY/DLL endpoint.[^pcie-us-if-rq-body][^pcie-us-axi-dma][^pcie-axi-master]
2. Vendor-specific wrappers (`pcie_us_if`, `pcie_s10_if`, `pcie_ptile_if`) translate proprietary stream formats and status signals into the same internal segmented TLP interface, enabling the shared DMA/AXI logic to operate unchanged across FPGA families.[^pcie-us-if][^pcie-s10-if][^pcie-ptile-if]
3. Example top-level designs instantiate proprietary PCIe HIP blocks but immediately terminate them at the AXI/AVST boundary, with Verilog-PCIe handling BAR apertures, MSI/MSI-X tables and configuration management.[^vcu118-top][^example-core-ptile][^pcie-msix]
4. Extensive simulation testbenches (cocotb/MyHDL) validate the adapters and DMA engines without relying on vendor models, underscoring that protocol functionality lives in the open-source codebase.[^test-pcie-us][^test-pcie-us-axi-dma]
5. Continuous integration executes these tests to guard against regressions, demonstrating an automated verification flow around the soft transaction layer.[^ci-workflow]

---

### Reference Links

[^pcie-us-if]: [`pcie_us_if.v`](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/pcie_us_if.v#L34-L493)
[^pcie-us-if-ports]: [`pcie_us_if.v` ports](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/pcie_us_if.v#L68-L207)
[^pcie-us-if-rc]: [`pcie_us_if` RC instantiation](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/pcie_us_if.v#L302-L337)
[^pcie-us-if-rq]: [`pcie_us_if` RQ parameters](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/pcie_us_if.v#L214-L260)
[^pcie-us-if-rq-body]: [`pcie_us_if_rq` TLP generation](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/pcie_us_if_rq.v#L177-L410)
[^pcie-us-if-rc-body]: [`pcie_us_if_rc` completion processing](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/pcie_us_if_rc.v#L169-L395)
[^pcie-s10-if]: [`pcie_s10_if.v`](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/pcie_s10_if.v#L34-L268)
[^pcie-ptile-if]: [`pcie_ptile_if.v`](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/pcie_ptile_if.v#L34-L280)
[^pcie-us-axi-dma]: [`pcie_us_axi_dma.v`](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/pcie_us_axi_dma.v#L34-L221)
[^pcie-us-axi-dma-rd]: [`pcie_us_axi_dma_rd.v`](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/pcie_us_axi_dma_rd.v#L183-L482)
[^pcie-us-axi-dma-fc]: [`pcie_us_axi_dma.v` credit wiring](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/pcie_us_axi_dma.v#L240-L331)
[^pcie-axi-master]: [`pcie_axi_master.v`](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/pcie_axi_master.v#L40-L357)
[^pcie-axil-master-minimal]: [`pcie_axil_master_minimal.v`](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/pcie_axil_master_minimal.v#L36-L248)
[^pcie-tlp-fifo]: [`pcie_tlp_fifo.v`](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/pcie_tlp_fifo.v#L34-L274)
[^dma-if-pcie-us]: [`dma_if_pcie_us.v`](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/dma_if_pcie_us.v#L37-L466)
[^vcu118-top]: [`example/VCU118/fpga/rtl/fpga.v`](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/example/VCU118/fpga/rtl/fpga.v#L230-L315)
[^example-core-pcie-us]: [`example_core_pcie_us.v`](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/example/common/rtl/example_core_pcie_us.v#L34-L268)
[^example-core-ptile]: [`example_core_pcie_ptile.v`](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/example/common/rtl/example_core_pcie_ptile.v#L210-L342)
[^example-core-s10]: [`example_core_pcie_s10.v`](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/example/common/rtl/example_core_pcie_s10.v#L193-L327)
[^pcie-us-msi]: [`pcie_us_msi.v`](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/pcie_us_msi.v#L34-L335)
[^pcie-msix]: [`pcie_msix.v`](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/pcie_msix.v#L33-L418)
[^pcie-us-if-config]: [`pcie_us_if.v` configuration capture](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/pcie_us_if.v#L493-L648)
[^pcie-us-if-mgmt]: [`pcie_us_if.v` management mux](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/pcie_us_if.v#L650-L843)
[^pcie-ptile-cfg]: [`pcie_ptile_cfg.v`](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/pcie_ptile_cfg.v#L37-L318)
[^pcie-s10-if-bar]: [`pcie_s10_if.v` BAR routing](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/rtl/pcie_s10_if.v#L120-L193)
[^test-pcie-us]: [`test_pcie_us.py`](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/tb/test_pcie_us.py#L1-L229)
[^test-pcie-us-axi-dma]: [`test_pcie_us_axi_dma_256.py`](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/tb/test_pcie_us_axi_dma_256.py#L1-L220)
[^test-dma-if-pcie-us]: [`test_dma_if_pcie_us_256.py`](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/tb/test_dma_if_pcie_us_256.py#L1-L210)
[^ci-workflow]: [GitHub Actions regression workflow](https://github.com/alexforencich/verilog-pcie/blob/25156a9a162c41c60f11f41590c7d006d015ae5a/.github/workflows/ci.yml#L1-L116)
