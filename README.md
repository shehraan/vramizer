<h1 style="margin-bottom: 0;">vramizer</h1>
<h4 style="margin-top: 4px;">A response to rising DRAM prices</h4>

<p align="center"> <img src="resources\imgs\pcPartPickerRam.png" alt="Skyrocketing RAM prices chart" width="1000"> </p>

As Desktop RAM and Storage prices have skyrocketed, one market has remained relatively stable - the used GPU market.

I myself was struggling to find good RAM for cheap, and that's when I had an idea: Why don't we try using VRAM to increase the overall system RAM in our computers?

It sounds simple really. The VRAM in high-end graphics cards today have a theoretical bandwidth of up to <b>1.79TB/s</b>!!! Even the best DDR5 RAM kits only go up to around 128 GB/s in dual channel-mode.

Knowing this, I thought that rather than going for the latest and greatest, it would be wise to save a buck and get a slower GPU instead, that still has VRAM matching desktop RAM speeds.

And so I bought a used AMD Radeon RX 570 with 8GB of VRAM for just $30 (a 8GB stick of desktop DDR5 RAM can cost $100+). And this one includes a GPU as a bonus too! Thus began my experiment to try to see if we could use VRAM to replace standard DRAM.

<p align="center"> <img src="resources\imgs\rx570.jpg" alt="AMD Radeon Rx 570 Graphics Card" width="1000"> </p>

## Background Information
Vramizer is a utility that uses the [FUSE library](http://fuse.sourceforge.net/)
to create a file system in VRAM. The idea is going to take advantage of system swap, which is when the computer offloads part of its programs to a secondary drive from RAM, usually caused by RAM overflow. Although, instead of going with the typical hard drives or SSDs, this program uses the video RAM of a discrete graphics card to store the extra files. 

An alternative approach using the kernel's MTD (Memory Technology Device) subsystem was explored first based on this [idea](https://web.archive.org/web/20081227185259/http://hedera.linuxnews.pl/_news/2002/09/03/_long/1445.html). Basically, MTD maps VRAM as a raw physical memory region via the PCIe BAR, exposing it as a block device without requiring OpenCL or a userspace daemon. However, benchmarking revealed that CPU-initiated reads across the PCIe BAR are significantly slower than OpenCL DMA (Direct Memory Access) transfers, producing over 5 times higher read/write latency compared to the FUSE-based system. Given that swap workloads are read-dominant, the FUSE-based approach proved superior in practice despite its additional complexity.

### Requirements
Linux with kernel 2.6+
A graphics card with support for OpenCL 1.2
<b> Note: </b> This program only works on Linux with an OpenCL-compatible GPU. Additionally, while I have tested it to work fine with Ubuntu and Arch, your mileage may vary with other distributions.

## Getting Started

First, install the OpenCL driver for your graphics card and verify that it's
recognized as an OpenCL device by running `clinfo`. Then install the `libfuse3-dev`
package or build it from source. You will also need `pkg-config` and OpenCL 
development files, (`opencl-dev`, `opencl-clhpp-headers` package or equivalent), 
with version 1.2 of the OpenCL headers at least.

Then, just run `make` to build `vramizer`.

Finally, create a directory that can be used as swap. In this case, you can just use a temporary path for it, if it's only meant to be used for experimentation. Then just simply mount a disk by running `bin/vramizer <mountdir> <size>`.
```
$ mkdir /tmp/vram #temporary directory for swap
$ bin/vramizer /tmp/vram 8G # Creates 8gb VRAM mountpoint
```

## Testing

### My System
- CPU: Ryzen 7 7700 (With integrated graphics, which I used for the display during my testing)
- GPU: RX 570 8GB
- OS: Arch Linux 2026.03.01
- Kernel Version: 6.18.13

### Benchmarking

I chose to compare this VRAM-based Desktop RAM implementation to my friend's DDR5 4800 CL40 kit. Below, is the command I used to test these two RAM options:

```
$ sudo fio \
--name=randrw4k \
--rw=randrw \
--rwmixread=70 \
--bs=4k \
--size=8G \
--iodepth=1 \
--runtime=30 \
--time_based \
--group_reporting \
--norandommap \
--randrepeat=0
```

### Results

| Metric | DDR5 4800 | vramizer |
|---|---|---|
| Read latency | ~80-100ns | 17,000ns |
| Write latency | ~80-100ns | 12,000ns |
| Read IOPS (4K random) | ~10,000,000+ | 43,800 |
| Write IOPS (4K random) | ~5,000,000+ | 18,800 |
| Read bandwidth | ~38 GB/s | 3.2 GB/s |
| Write bandwidth | ~38 GB/s | 2.74 GB/s |
| Latency vs DDR5 | 1x (baseline) | ~50-100x slower |

While the results disproved my initial theory that VRAM would be faster, it was not entirely unexpected. 

As I learned more about how the GPU and its VRAM interacted with the CPU, I began to realize the physical limitations in my path. I've attached some of my main reflections below.

## Learnings

#### 1. Theoretical bandwidth is not the same as real-world bandwidth
Theoretical bandwidth of VRAM usually refers to the direct connection between the GPU and its VRAM.

To use the VRAM as system RAM / swap however, it needs to take a much longer path from:
VRAM --> GPU --> PCIE Lanes --> Chipset (if applicable) --> RAM/Cache --> CPU

The long pathway, along with the limitations of its compoentns (such as the max bandwidth of PCIE Lanes themselves, which the Graphics card is plugged into), introduces extreme overhead which strongly affects speeds and latency.

#### 2. “Swap on VRAM” is not one thing
It’s best separated it into 3 different buckets:

I. Real Linux swap
- Needs a block device or swap file
- Works with things like SSDs, partitions, zram, swapfiles

II. MTD/phram hack: my first attempt at getting to work

- Maps a physical address range and pretends it is an MTD device
- Then mtdblock makes it look block-like enough for swap tools
- Has a <i>256MB limit</i>, as that's the max BAR size.
- Very cursed, very old-school

III. FUSE / OpenCL

- Uses GPU buffers through OpenCL
- Exposes a filesystem in userspace
- Then puts a swapfile on top of that
- Still cursed, but much more practical for multi-GB use

#### 3. PCIe interface asymmetry matters

Writes → fast (instantly posted)

Reads → slow (blocking caused by round trip delay while waiting for response from GPU)

Swap = read-heavy → pain

#### 4. Performance Depends on:

“Are we doing CPU cache-driven reads?” (bad)

vs.

“Are we doing bulk/DMA transfers?” (good)

#### 4. “FUSE is slow, so it must be worse than MTD” is wrong (context-dependent).

FUSE overhead = microseconds

PCIe MMIO reads = much worse bottleneck

If program uses DMA → it can beat MTD

## Conclusion
If your goal is just experimenting the best value test card for this might just be a used RX 570 8GB. It worked incredibly well for this experiment and I'm definitely going to be keeping it for future use. 

However, if your goal is actually improving system behavior, not just experimenting:
- use zram
- maybe zswap

Do not rely purely on VRAM swap hacks unless you must. It is not intended for serious use, though it does actually work fairly well, especially since consumer GPUs with 8GB or more VRAM are now readily (and cheaply) available.
