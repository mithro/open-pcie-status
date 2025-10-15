# Open PCIe Status Report

This repository collects an AI-generated multi-chapter report on the state of open-source PCI Express (PCIe) development across FPGA platforms.

## Chapter index
- [Chapter 1 – Introduction and Protocol Overview](chapter1.md): outlines the layered architecture of PCIe and USB 3.x, highlights shared physical mechanisms such as 8b/10b and 128b/130b encoding, and explains how the PIPE interface enables protocol-agnostic PHY reuse.
- [Chapter 2 – Relationship Between RTL, Hard Blocks, PHYs and Open Toolchains](chapter2.md): explains the trade-offs between soft logic and hardened IP, examines internal versus external PHY integration, and surveys toolchain considerations for open implementations.
- [Chapter 3 – Survey of Open Projects and Protocol Coverage](chapter3.md): catalogs community PCIe and USB projects, mapping which protocol layers they implement and the FPGA families and toolchains they target.
- [Chapter 4 – Project Analyses: States, Successes and Challenges](chapter4.md): provides in-depth case studies of prominent PCIe/USB cores, summarizing their goals, achievements, limitations, and readiness for fully open stacks.
- [Chapter 5 – Toward a Fully Open PCIe Core: A Roadmap](chapter5.md): proposes guiding principles and a modular architecture roadmap that combines open PHY, data link, and transaction layer efforts supported by open tooling.

Use this index to jump into chapters of interest or to follow the narrative from foundational concepts to implementation challenges and future directions.

## LitePCIe Analysis Reports

In-depth technical analyses of the LitePCIe project:
- [LitePCIe Architecture Analysis (Claude Opus 4.1)](litepcie-analysis-claude-opus-4-1.md): examines the boundary between LitePCIe's custom implementation and vendor-supplied IP blocks.
- [LitePCIe Architecture Analysis (ChatGPT-5)](litepcie-analysis-chatgpt5.md): documents which protocol layers are implemented inside LitePCIe and which functions depend on proprietary hard IP.
