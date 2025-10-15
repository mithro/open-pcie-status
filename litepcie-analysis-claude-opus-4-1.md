# LitePCIe Architecture Analysis (Claude Opus 4.1)

This document analyzes what functionality LitePCIe implements versus what relies on vendor-supplied IP blocks, based on examination of the source code and documentation.

## Table of Contents

- [Core Architecture](#core-architecture)
- [Vendor PHY Interface Definition](#vendor-phy-interface-definition)
- [LitePCIe Custom Implementation](#litepcie-custom-implementation)
  - [TLP Processing Layer](#tlp-processing-layer)
  - [DMA Engine](#dma-engine)
  - [Interrupt Controllers](#interrupt-controllers)
  - [Crossbar](#crossbar)
  - [Clock Domain Crossing](#clock-domain-crossing)
- [Vendor-Specific Implementations](#vendor-specific-implementations)
  - [Xilinx 7-Series Implementation](#xilinx-7-series-implementation)
  - [Xilinx UltraScale/UltraScale+ Implementation](#xilinx-ultrascaleultrascale-implementation)
  - [Intel Cyclone V Implementation](#intel-cyclone-v-implementation)
  - [Lattice CertusPro-NX Implementation](#lattice-certuspro-nx-implementation)
- [Comparative Analysis](#comparative-analysis)
  - [Functionality Comparison Table](#functionality-comparison-table)
  - [PCIe Enumeration Division](#pcie-enumeration-division)
  - [Implementation Boundaries](#implementation-boundaries)
    - [Physical Interface Boundary](#physical-interface-boundary)
    - [Layer Responsibility Division](#layer-responsibility-division)
    - [Data Flow Across the Boundary](#data-flow-across-the-boundary)
    - [Portability Through Abstraction](#portability-through-abstraction)
    - [Current Implementation Division](#current-implementation-division)

## Core Architecture

LitePCIe implements its Transaction Layer Protocol (TLP) processing entirely in custom Migen/Python code while relying on vendor hard IP blocks for the Physical and Data Link layers. The boundary between these is defined by a simple streaming interface.

## Vendor PHY Interface Definition

The interface between vendor IP and LitePCIe is defined in [`litepcie/common.py`](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/common.py#L290-L294):

```python
def phy_layout(data_width):
    layout = [
        ("dat", data_width),      # Raw TLP data bytes
        ("be",  data_width//8)    # Byte enables
    ]
    return EndpointDescription(layout)
```

This streaming interface carries raw TLP bytes from the vendor PHY to LitePCIe's custom logic. All TLP intelligence resides in LitePCIe's implementation above this boundary.

## Vendor-Specific Implementations

### Xilinx 7-Series Implementation

The Xilinx 7-Series PHY wrapper ([`litepcie/phy/s7pciephy.py`](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/phy/s7pciephy.py)) uses these Xilinx hard IP blocks:

#### PCIE_2_1 Hard Block
The [PCIE_2_1](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/phy/s7pciephy.py#L146-L228) primitive implements:
- PCIe Physical Layer including SERDES
- Data Link Layer with ACK/NAK protocol
- Link Training and Status State Machine (LTSSM)
- Configuration space registers (0x00-0xFF)
- Flow control credit management
- DLLP generation and processing
- TLP CRC generation/checking

Configuration parameters are set via TCL ([lines 474-531](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/phy/s7pciephy.py#L474-L531)):
```python
config = {
    "Component_Name": ip_name,
    "Vendor_ID": "0x10EE",
    "Device_ID": "0x7021",
    "Bar0_Scale": "Megabytes",
    "Bar0_Size": bar0_size//1024//1024,
    "Link_Speed": {64: "2.5_GT/s", 128: "5.0_GT/s"}[pcie_data_width],
    "Maximum_Link_Width": f"X{nlanes}",
}
```

#### Clock Resources
- [IBUFDS_GTE2](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/phy/s7pciephy.py#L127-L132): Differential clock input buffer for PCIe reference clock
- [BUFG](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/phy/s7pciephy.py#L136): Global clock buffer
- [S7MMCM](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/phy/s7pciephy.py#L413-L436): Mixed-Mode Clock Manager for generating user clocks

### Xilinx UltraScale/UltraScale+ Implementation

The UltraScale PHY ([`litepcie/phy/uspciephy.py`](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/phy/uspciephy.py)) uses more advanced hard blocks:

#### PCIe Hard Block Variants
Selection logic at [lines 175-185](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/phy/uspciephy.py#L175-L185):
```python
pcie_hard_block = {
    "gen3" : {
        False : "PCIE3_UltraScale",
        True  : "PCIE30E_UltraScale",
    },
    "gen4" : {
        False : "PCIE40E_UltraScale",
        True  : "PCIE4CE_UltraScale",
    }
}[speed][is_versal]
```

These blocks implement:
- Gen3 (8.0 GT/s) or Gen4 (16.0 GT/s) speeds
- Up to x16 lane configurations
- Advanced equalization for Gen3/Gen4
- Extended configuration space support
- Enhanced flow control

### Intel Cyclone V Implementation  

The Intel PHY ([`litepcie/phy/c5pciephy.py`](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/phy/c5pciephy.py)) uses:

#### altpcie_hip Hard IP
Instantiated via TCL at [lines 374-401](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/phy/c5pciephy.py#L374-L401):
- Implements PCIe Gen1/Gen2 Physical and Data Link layers
- Uses Avalon-ST interface instead of AXI
- Requires protocol conversion wrappers

LitePCIe provides [AvalonST2Native](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/phy/c5pciephy.py#L68-L111) and [Native2AvalonST](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/phy/c5pciephy.py#L115-L158) converters to translate between Intel's Avalon-ST and LitePCIe's native protocol.

### Lattice CertusPro-NX Implementation

The Lattice PHY ([`litepcie/phy/lfcpnxpciephy.py`](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/phy/lfcpnxpciephy.py)) uses:

#### pcie_hardblock_x1 IP
Configured at [lines 312-335](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/phy/lfcpnxpciephy.py#L312-L335):
- PCIe Gen2 x1 hard IP implementation
- Integrated SERDES and PCIe controller
- Limited to single lane operation

## LitePCIe Custom Implementation

### TLP Processing Layer

LitePCIe implements complete TLP handling in [`litepcie/tlp/`](https://github.com/enjoy-digital/litepcie/tree/master/litepcie/tlp):

#### TLP Depacketizer
[`litepcie/tlp/depacketizer.py`](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/tlp/depacketizer.py) ([lines 126-406](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/tlp/depacketizer.py#L126-L406)):
- Parses raw TLP bytes from PHY stream
- Extracts header fields (fmt, type, length, requester ID, tag)
- Routes packets based on type (memory, I/O, configuration, completion)
- Handles both 3DW and 4DW headers

#### TLP Packetizer  
[`litepcie/tlp/packetizer.py`](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/tlp/packetizer.py) ([lines 122-363](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/tlp/packetizer.py#L122-L363)):
- Constructs TLP headers from request parameters
- Manages data alignment and byte enables
- Implements both 32-bit and 64-bit addressing

#### TLP Controller
[`litepcie/tlp/controller.py`](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/tlp/controller.py) ([lines 87-152](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/tlp/controller.py#L87-L152)):
- Tracks pending requests with tag management
- Implements completion timeout detection
- Manages maximum pending request limits

### DMA Engine

The DMA implementation in [`litepcie/frontend/dma.py`](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/frontend/dma.py) provides:

#### Scatter-Gather DMA
[LitePCIeDMAScatterGather](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/frontend/dma.py#L31-L102):
- Hardware descriptor processing
- Loop mode for continuous operation
- Prog mode for software control

#### Descriptor Splitter
[LitePCIeDMADescriptorSplitter](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/frontend/dma.py#L106-L189):
```python
"""Splits descriptors from LitePCIeDMAScatterGather in shorter descriptors of:
- Maximum Payload Size for Writes.
- Maximum Request Size for Reads.
"""
```

### Interrupt Controllers

Three MSI implementations in [`litepcie/core/msi.py`](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/core/msi.py):

- [LitePCIeMSI](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/core/msi.py#L13-L55): Single MSI vector
- [LitePCIeMSIMultiVector](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/core/msi.py#L59-L126): Up to 16 MSI vectors  
- [LitePCIeMSIX](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/core/msi.py#L130-L219): MSI-X with up to 64 vectors

### Crossbar

The [LitePCIeCrossbar](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/core/crossbar.py#L14-L222) implements:
- Multi-master to multi-slave routing
- Channel-based packet steering
- Request arbitration
- Completion routing

### Clock Domain Crossing

Each PHY implements clock domain crossing to isolate the PCIe clock from user logic. For example, in [s7pciephy.py](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/phy/s7pciephy.py#L283-L297):

```python
# TX (fpga --> host)
self.tx_cdc = stream.ClockDomainCrossing(
    layout         = tlp_raw_layout(data_width),
    cd_from        = cd,
    cd_to          = "pcie",
    buffered       = True,
    with_common_rst = True
)
```

## Comparative Analysis

### Functionality Comparison Table

| Functionality | Vendor Hard IP | LitePCIe | Notes |
|---|---|---|---|
| **SERDES/PHY** | ✓ | ✗ | Multi-gigabit transceivers |
| **8b/10b encoding** | ✓ | ✗ | Hardware encoder/decoder |
| **LTSSM** | ✓ | ✗ | Link training state machine |
| **DLLP handling** | ✓ | ✗ | Data Link Layer packets |
| **ACK/NAK protocol** | ✓ | ✗ | Link-level acknowledgments |
| **Flow control credits** | ✓ | ✗ | VC credit management |
| **Standard config space (0x00-0xFF)** | ✓ | ✗ | Device/Vendor ID, BARs |
| **Extended config space (0x100-0xFFF)** | ✗ | ✓ | [Handled by LitePCIe](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/core/endpoint.py#L168-L175) |
| **TLP packet construction** | ✗ | ✓ | [LitePCIeTLPPacketizer](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/tlp/packetizer.py) |
| **TLP packet parsing** | ✗ | ✓ | [LitePCIeTLPDepacketizer](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/tlp/depacketizer.py) |
| **Memory transaction handling** | ✗ | ✓ | BAR address decoding |
| **DMA engines** | ✗ | ✓ | [Scatter-gather DMA](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/frontend/dma.py) |
| **MSI/MSI-X generation** | ✗ | ✓ | [Interrupt controllers](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/core/msi.py) |
| **Request/completion routing** | ✗ | ✓ | [Crossbar implementation](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/core/crossbar.py) |
| **Flow control (TLP level)** | ✗ | ✓ | [Pending request tracking](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/tlp/controller.py) |
| **Protocol bridges** | ✗ | ✓ | [AXI](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/frontend/axi.py), [Wishbone](https://github.com/enjoy-digital/litepcie/blob/master/litepcie/frontend/wishbone.py) |
| **Clock domain crossing** | ✗ | ✓ | AsyncFIFO between pcie/sys domains |
| **Data width conversion** | ✗ | ✓ | 64/128/256/512-bit support |

### PCIe Enumeration Division

During PCIe enumeration, the responsibilities are divided:

**Vendor IP Handles:**
- Recognition of Type 0 configuration TLPs based on Bus/Device/Function
- Storage of standard configuration registers (Device ID, Vendor ID, Class Code, BARs)
- Automatic generation of completion TLPs for standard config space reads
- Implementation of power-up unconfigured state

**LitePCIe Handles:**
- Extended configuration space transactions (0x100-0xFFF)
- BAR address decoding after enumeration
- Memory-mapped transaction processing
- MSI-X table and pending bit array management

The division is clear: when the host reads offset 0x00 (Device ID), the vendor IP responds directly without LitePCIe involvement. When the host accesses memory within a BAR range, the vendor PHY passes the TLP to LitePCIe for processing.

### Implementation Boundaries

#### Physical Interface Boundary

The interface between vendor IP and LitePCIe is a streaming protocol carrying raw TLP data bytes. This boundary is defined by the `phy_layout()` function which specifies only two signals per direction: data bytes and byte enables. All TLP intelligence resides above this interface in LitePCIe's custom logic.

#### Layer Responsibility Division

The architectural boundary cleanly separates PCIe protocol layers:

**Vendor Hard IP Implements:**
- Physical Layer (PHY)
  - SERDES transceivers
  - 8b/10b or 128b/130b encoding/decoding
  - Clock and data recovery
  - Equalization and signal conditioning
- Data Link Layer (DLL)
  - DLLP generation and processing
  - ACK/NAK protocol
  - TLP CRC generation and checking
  - Retry buffer management
  - Link Training and Status State Machine (LTSSM)
- Base Configuration Space (offsets 0x00-0xFF)
  - Device/Vendor ID registers
  - BARs, Capabilities pointer
  - Standard config space read/write handling

**LitePCIe Implements:**
- Transaction Layer (TLP)
  - TLP header construction and parsing
  - Request/Completion matching
  - Tag management and allocation
  - Completion timeout detection
  - Reordering buffer
- Application Layer Features
  - DMA engines with scatter-gather support
  - MSI/MSI-X interrupt generation
  - BAR address decoding and routing
  - Memory and I/O transaction processing
  - Extended configuration space (0x100-0xFFF)
- System Integration
  - Clock domain crossing
  - Data width conversion (64/128/256/512-bit)
  - Protocol bridges (AXI, Wishbone, Avalon)
  - Crossbar arbitration

#### Data Flow Across the Boundary

**Receive Path:**
1. Vendor PHY receives TLPs from the PCIe link
2. DLL validates CRC and handles retries
3. Raw TLP bytes stream into LitePCIe via `source.dat` and `source.be`
4. LitePCIe depacketizer extracts headers and routes to appropriate handlers
5. Application logic processes requests and generates responses

**Transmit Path:**
1. LitePCIe application logic generates transaction requests
2. Packetizer constructs TLP headers with proper addressing and tags
3. Formatted TLPs stream to vendor PHY via `sink.dat` and `sink.be`
4. DLL adds CRC and handles link-layer protocol
5. PHY transmits on PCIe lanes

#### Portability Through Abstraction

This clean boundary enables LitePCIe to achieve vendor independence:

- **Same core logic** runs on Xilinx, Intel, Lattice, and Gowin FPGAs
- **Vendor-specific PHY wrappers** only translate between native protocols (AXI-Stream, Avalon-ST, etc.) and the unified `phy_layout()` interface
- **Application code** remains unchanged across platforms
- **Protocol behavior** (tag allocation, completion reordering, MSI generation) is consistent regardless of underlying hard IP

#### Current Implementation Division

The boundary shows how functionality is currently divided:

**Currently Implemented in Vendor Hard IP:**
- SERDES hard macros (physical signaling)
- LTSSM implementation (link training sequences)
- Flow control credit counters (DLL state machine)
- TLP CRC generation and checking
- Retry buffer management

**Currently Implemented in LitePCIe Open Source:**
- All Transaction Layer processing
- DMA engines and memory controllers
- Interrupt generation and routing
- System bus interfaces and bridges
- Application-specific packet processing

This architecture shows that LitePCIe implements the majority of PCIe functionality—everything above the Data Link Layer—in portable, open-source HDL. The vendor IP provides the physical connectivity and base protocol compliance, while LitePCIe implements all application behavior, packet intelligence, and transaction processing.
