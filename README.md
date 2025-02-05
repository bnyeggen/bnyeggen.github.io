I build AI / machine learning systems.

Email: [bryce@nyeggen.com](bryce@nyeggen.com)

LinkedIn: [https://www.linkedin.com/in/bryce-nyeggen-73559438/](https://www.linkedin.com/in/bryce-nyeggen-73559438/)

#### Blog

##### Big Iron for local LLM development
*February 5, 2025*

Large language models have some interesting characteristics that affect hardware choices for local development.  The first is that they actually exist - in the last few months, the capabilities of open models like Llama, DeepSeek v3, and Phi 4 have absolutely exploded.  It is still overwhelmingly cheaper to run them via APIs (DeepSeek v3 for instance costs a whopping $0.28 per million output tokens on its native platform), but for research purposes, when privacy is a priority, or when it can be used for multiple projects, there can be a good case for running locally on relatively inexpensive hardware.

For a long time the meta for local inference has been a pair of Nvidia RTX 3090, on whatever consumer gaming hardware was handy.  They are still the cheapest way to get 24 GB of VRAM in a single GPU - about $800 on eBay at time of writing.  Consumer full-size ATX motherboards tend to have two PCIe slots for GPUs, so a pair of 3090s maximizes this while falling inside the power budget of a consumer gaming power supply.  There are professional and datacenter GPUs with more VRAM per card, but they are vastly more expensive per GB.

VRAM capacity, and to a lesser extent GPU memory bandwidth, is the dominating priority for local inference due to the operations these models perform.  Typically, an open model is converted to a format that uses reduced-precision integer operations instead of the higher-precision floating point operations it was trained with.  The goal is to shrink the size of the model in memory, but the secondary effect is that they use surprisingly little actual compute.  Mostly the GPU spends its time scanning over model parameters.  This scanning leverages high GPU internal memory bandwidth, but only over data that actually fits on the card.  

We can use multiple cards, or multiple memory subsystems, but because most models are structured as a series of sequential layers, we end up running first one set of layers on one device, then the next, etc.

To play this out, here is the relative bandwidth of various subsystems.

- PCIe 3.0 x16 bus: 16 GB/s
- PCIe 4.0 x16 bus: 32 GB/s
- DDR4-2666 RAM: 21.3 GB/s per channel
- DDR5-4800 RAM: 38.4 GB/s per channel
- Nvidia RTX 3090: 936.2 GB/s (internal, over 24gb of VRAM)
- Nvidia RTX A4000: 448 GB/s (internal, over 16gb of VRAM)

I claim that the “3090 meta” is broken, and you can get more capabilities at a lower price by using last-generation workstation hardware and professional GPUs.  Observe:

This is a HP Z8 G4.  I bought it as a relatively barebones system from eBay, along with 90% of the other hardware.  Here is the complete BOM:

- HP Z8 G4 (2x Xeon Gold 6136 CPUs): $600
- 768GB DDR4-2666 ECC RDIMM RAM: $645
- Leftover storage devices and wifi: ~free
- 4x Nvidia A4000 16gb: $450 each

This has a few quirks versus consumer hardware.  The most notable is that it has two CPUs, each with their own memory controller, and each capable of using 6 channels of DDR4 RAM.  That is 12 channels total, giving us a theoretical memory bandwidth of 255.6 GB/s, or half that (127.8 GB/s) per CPU.  Each CPU has preferential access to its own local RAM and slower access to memory connected to the other CPU, so in the worst case it makes sense to model them as two separate devices, each with its own 384 GB of RAM (a detailed answer would get into arcana of things like NUMA architectures and what specific tensor operations can be partitioned rather than pipelined between devices).

The second quirk is that it has quite a number of PCIe lanes and slots - 48 lanes and two physical full-speed x16 PCIe 3.0 slots per CPU, plus a number of lanes available for secondary slots.  Consumer hardware typically would end up running two cards max per system at reduced x8 bandwidth.

The net effect of this is we have *a lot* of memory and bandwidth on the cheap.  Instead of two 3090s, giving us in aggregate 48 GB of VRAM at about $1600 in GPUs, we have 64 GB for about $1800 - a good deal, given the capability bump.  We also have 768 GB of system RAM, which lets us run very large open models that do not fit in GPU only.

Of course, this is last-generation hardware, so we are using PCIe 3.0 and DDR4 memory, rather than PCIe 5.0 and DDR5.  Nevertheless, the number of memory channels and PCIe lanes makes aggregate bandwidth comparable to current-generation hardware, at much higher capacities, and at a vastly lower price.  512 GB of “enterprise” (ECC RDIMM) DDR5 RAM alone approaches the price of our entire system, and to run it you would need current (ie, even more expensive) enterprise hardware.  On consumer 4-slot systems, you can’t get more than 192 GB of DDR5 at any price.

On DeepSeek-R1, I average about 3.1 tokens per second in llama.cpp, on a machine that (disregarding the GPUs) cost under $1300.  The GPUs do not currently accelerate that particular model (although they may if the internal representation is changed) - they will however be useful for other projects.  As a teaser, note that our aggregate GPU internal memory bandwidth is equivalent to a pair of 3090s, the cost is 12.5% higher, but the VRAM capacity is 33% higher (allowing us to run 70B class models completely in GPU at acceptable quantization), and the PCIe bandwidth is a full 2x that of a newer consumer system splitting x16 total PCIe 4.0 lanes across two cards.
