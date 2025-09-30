# inference-benchmarks
Some benchmarks for LLM inference and some notes on building your own.

# Testing

- [Snowpiercer 15B MultiGPU test](multi-gpu.md)
    - Are multiple GPUs faster without tensor parallel?

- [Performance Updates](performance-updates.md)
    - Recent PRs have been merged into llama.cpp which should improve performance for GFX906 among other cards. How much?

- [MI50 vs 3090](mi50-vs-3090.md)
    - Just how much slower is the MI50 than everyone's favourite card, anyway?

- DeepSeek 3.1 Q4_1
    - Adventures into cramming DeepSeek 3.1 Q4_1 into the system and how to get "usable" performance out of it.

- GLM4.5 Q4_1
    - Given it's a smaller model, we should get better performance. How fast does it run and can we use the tricks we learned last time?

- Qwen3 235B A22B
    - Do the tricks work here either?

- DeepSeek 3.1 Q2_K_XL
    - This actually fits on the cards, so it should be faster than Q4_1, yet it isn't? Why?

# The Hardware

- **Platform**: LGA2066
- **Motherboard**: MSI X299-A Pro ("E7A94")
- **BIOS config**:
    - "Above 4G memory" - Enabled
    - "MMIO High Granularity" - 256GB
    - "Intel iRST" off. Will enable PCIe bifurcation on slot but will break everything if it sees a device that's not a M2 drive. More below.
- **CPU**:
    - i9 10900X
    - Not at all similar to the i9 10900K, this one is a badge engineered 7th gen, and actually has significantly worse single thread performance.
    - But unlike the 10900K, you get 4 ram channels and AVX512.
- **RAM**:
    - 256GB - 8x 32GB Patriot Viper Steel PVS464G360C8K
    - Supposedly rated to 3600mhz but I'm yet to see it. Can barely tighten the timings at 2933. But I am using four 2-stick kits, not a matched set of 8.
- **Devices** :
    - **Onboard LAN**: disabled, seems to have died.
    - **M2_1**: Samsung 970 1TB
    - **M2_2**: Samsung 980 500GB
    - **PCI_E1 x16**: PLX8749 with 4x MI50 (x8 PCIe 3.0 each)
    - **PCI_E2 x4**: Mellanox ConnectX-4 LX with 10GBe to LAN, and 25GBe direct to desktop PC
    - **PCI_E3 x16**: PLX8749 with 4x MI50 (x8 PCIe 3.0 each)
    - **PCI_E4 x1**: None
    - **PCI_E5 x8**: PLX88024 with 4x M.2 slots.
        - Samsung 980 500GB
        - Intel 665p 1TB
        - Intel P4510 2TB U.2 on M.2 adaptor
        - Nvidia RTX3090 on M.2 to Oculink breakout
    - **U2_1**: left empty as it is shared with PCI_E2
- **KVM**:
    - Raspberry Pi 4 running [DiY PiKVM v2](https://docs.pikvm.org/v2/)
    - [UGreen USB HDMI capture card](https://www.amazon.com.au/dp/B0DGXJS6BF?ref=ppx_yo2ov_dt_b_fed_asin_title)
    - Relay breakout on GPIO to power button
    - [Motherboard header to USB-C cable](https://www.aliexpress.com/item/1005008672546194.html?) with red wire removed
        - Pi 4 can be a usb device via the USB-C port, can appear to PC as mouse/kbd/disk/network
    - Powered from 5VSB on second PSU, through [Wide Input Shim](https://shop.pimoroni.com/products/wide-input-shim?variant=2168104321034) (not strictly needed but will cope with voltage drop better)

# Software
- OS: Debian 12.1
- ROCm version: 6.4.3 (instructions [here](https://github.com/ROCm/ROCm/issues/4625#issuecomment-2934325443) to make it work with MI50)

# Fans
- Modified "squirrel cage" fans inserted into end of MI50.
- These are 12v 4-wire PWM fans.
- All running off one fan output via adaptors that connect to molex for 12v power, but take PWM signal from motherboard fan header.
- Python script monitors GPU temps, and scales fans / adjusts power limit to allow lower fan speeds.
- Will at some point try cooling with two 120mm 3000rpm Noctua IndustrialPPC fans. May not be sufficient airflow for the cards.
- The 3090 had a dead onboard fan - deshrouded, and put two Noctua NF-A12x25 fans on it.

# A bit more about the PCIe hardware
- Originally had bifurcation set up.
    - ADT PCIe to MCIO risers - 1x16 to 2x8
    - Supplied MCIO cables are a bit loose, will randomly boot up as x4 if they're bumped off axis. I don't have this problem with any other cables.
    - How to enable bifurcation in a BIOS that does not offer it: see [here](https://www.reddit.com/r/Amd/comments/14bnqh3/guide_about_how_to_check_pcie_bifurcation_support/) and [here](https://winraid.level1techs.com/t/guide-how-to-bifurcate-a-pci-e-slot/32279).

- PLX8749 switch card to 4x SFF-8654 (SlimSAS) ports. Switches on card control whether it's two x16 links (use two ports together), x8 (use each port), x4 (two devices on each port).
- SlimSAS to MCIO cables between the card and breakouts. A mixture of 50cm, 75cm and a 1m cable. With decent cables, 1m seems to work fine even though ribbon cable risers >20cm are problematic.
- ADT F37B PCIe breakouts.
    - Recommend not buying these. MCIO pinout is mirrored!
    - I have fixed this by cutting up the SlimSAS to MCIO cable.
        - You need to move PERST# to the other side and ground CPRSNTA#. It's not a high speed signal so you'll be fine.
        - You need to move REFCLK to the other side. Polarity doesn't matter, and it's only 100Mhz, so hand solder bodging is fine.
        - You do not need to move PCIe wires around because luckily all the lanes still connect when pinout is mirrored. They are in reverse order but PLX supports lane reversal (this is a fairly standard PCIe feature for board layout reasons), and polarity reversal (support is mandatory in PCIe spec) - so it just works.
        - Don't cut the wrong wire, I wouldn't rate your chances of PCIe working over a splice, but equally I wouldn't be surprised if it did. Will have to try on the one I did this to.
    - Potential alternative riser: [this one](https://www.aliexpress.com/item/1005008340538991.html)
        - This one uses REFCLK and PERST# from channel B, but unlike the ADT riser, the pinout is not mirrored. It works on my PLX switch cards as they output those signals to A and B even in x8 mode. It will likely not work on a passive breakout.
- PLX88024 card in the x8 slot
    - This one is a "no bifurcation required" PCIe 4.0 x8 to 4x M.2 adaptor.
    - The motherboard/CPU is only PCIe 3.0 but it can still talk PCIe 4.0 to the SSDs.
    - Due to some poor prior planning, none of the SSDs ended up being PCIe 4.0.
    - It is also running the 3090 via a M.2 to Oculink adaptor. It does not have a problem with this not being a SSD. PCIe 4.0 x4 is also the same bandwidth as the upstream PCIe 3.0 x8.

# Building cases from T-Slot

TODO: write more about this.
