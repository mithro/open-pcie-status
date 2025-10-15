# Introduction and Protocol Overview

## Layered Architecture of PCIe

PCI Express (PCIe) is organised as a layered protocol stack. The **Physical Layer** handles the electrical link: it trains the link, sets the speed and lane width, and manages lane numbering[[1]](sources.md#source1). Above it, the **Data Link Layer (DLL)** ensures reliability by sequencing transaction‑layer packets, generating a link cyclic redundancy check (LCRC), sending acknowledgements (ACK) or negative acknowledgements (NAK) via Data Link Layer packets (DLLPs), and managing credit‑based flow control[[1]](sources.md#source1). The **Transaction Layer** creates and consumes Transaction Layer Packets (TLPs); it provides memory read and write requests, completions, and messages that carry payloads or control information[[2]](sources.md#source2).

## Layered Architecture of USB 3.x

USB 3.x introduces a similar layered model. The **Physical Layer** provides bit‑level transmission over differential pairs using 8b/10b encoding at 5 Gb/s (SuperSpeed) and 128b/132b at higher rates. On top of it, the **Link Layer** maintains link connectivity, manages physical layer events (connect, remove, power management), performs flow control and reliable delivery of packet headers, frames outgoing packets, buffers data, and handles link errors[[3]](sources.md#source3). The **Protocol Layer** implements host‑to‑device communications. It defines transaction packets (TPs), data packets (DPs) and link management packets (LMPs), schedules transfers, and uses NRDY/ERDY handshakes for asynchronous communication[[3]](sources.md#source3). A final **Device Layer** implements class‑specific behaviour for endpoints.

## Shared Foundations and the PIPE Interface

Despite their different purposes, PCIe and USB 3.x share fundamental mechanisms. Both use serial transceivers with embedded clock recovery and bit‑scrambling; early generations use **8b/10b encoding**, while later versions move to **128b/130b** or **128b/132b** encoding. Both rely on link‑training sequences to negotiate speed and lane width and use credit‑based flow control.

    The **PHY Interface for the PCI Express (PIPE)** standard provides a common interface between the digital controller (MAC) and the physical transceiver. It specifies transmit and receive parallel buses, out‑of‑band signalling for link training and low‑power states, and control/status lines for reset and power management. Because PIPE is protocol‑agnostic, the same transceiver wrapper can support PCIe, USB 3.x, SATA or DisplayPort by swapping the digital state machine. This makes it possible to design a common open‑source physical layer that supports multiple protocols.

![Protocol stack comparison]({{file:file-8rumspCd7SJPRE2rTDm6d3}})

*Figure 1: Conceptual comparison of PCIe and USB 3.x protocol stacks highlighting shared physical layers and the PIPE interface. Both protocols share common physical mechanisms such as 8b/10b encoding and lane training; the PIPE interface allows reusing PHYs across protocols.*

## Similarities and Differences

The major differences between PCIe and USB 3.x lie at the **transaction/protocol layer**. PCIe uses a peer‑to‑peer model where endpoints can initiate memory transactions and completions, while USB employs a host‑directed model in which devices respond to host requests and cannot autonomously initiate traffic. PCIe TLPs include memory, configuration, and message requests; USB SuperSpeed uses transactions built from data packets, transaction packets, and link management packets[[3]](sources.md#source3). At the link layer, PCIe’s credit‑based flow control and ACK/NAK mechanism are similar in concept to USB’s NRDY/ERDY handshake but have different timing and packet semantics.

This overview sets the stage for examining how open‑source projects implement these layers and how shared building blocks can accelerate development of fully open PCIe and USB 3.x cores.