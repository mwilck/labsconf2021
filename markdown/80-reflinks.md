<!-- .slide: data-state="divider" id="divider" data-timing="20s" data-menu-title="Divider with image" -->
# Initramfs Reflinks

<a title="By Brandenads, Public Domain" href="https://en.wikipedia.org/wiki/Tetris#/media/File:Typical_Tetris_Game.svg">
    <img alt="Tetris" src="images/Typical_Tetris_Game_wik_public_domain_brandenads.svg"/>
</a>


<!-- .slide: data-state="normal" id="reflinks-intro" data-timing="20s" data-menu-title="Reflinks Introduction" -->
# Reflinks

*   Avoid duplication of cpio archive data
    *   Extent sharing between initramfs source files and cpio archive
*   Metadata I/O
    *   cpio compression, symbol-strip and data copy can be skipped
*   Can be used alongside kernel module compression


<!-- .slide: data-state="normal" id="reflinks-caveats" data-timing="20s" data-menu-title="Reflinks Caveats" -->
# Reflinks: Restrictions and Limitations

*   Btrfs or XFS filesystems only
*   cpio archiver must perform reflink aware file I/O
    *   `copy_file_range()` or `FICLONERANGE`
*   Sensitive to alignment
    *   Complicated by cpio header requirements
*   Initramfs sources, staging and destination paths must share a mount point
*   Causes significant fragmentation


<!-- .slide: data-state="normal" id="reflinks-boot" data-timing="20s" data-menu-title="Reflinks Boot" -->
# Reflinks: Boot Considerations

*   Bootloader must be capable of reading dracut-cpio initramfs archives
    *   GRUB Btrfs broken by gaps between extents (fix from Qu)
*   Kernel cpio decompression can be avoided
*   Large and fragmented initramfs images may outweigh performance gains
*   Patchset: disable kernel `INITRAMFS_PRESERVE_MTIME`


<!-- .slide: data-state="normal" id="reflinks-impl1" data-timing="20s" data-menu-title="Reflinks Implementation 1" -->
# Archiver Implementation: GNU cpio

*   First implementation with help from Luis Henriques
*   `copy_file_range()` in copy-out code path
    *   New `--reflink` and `--chain` parameters
*   Reflink alignment provided via separate `padcpio` utility
    *   Inject sparse files into Dracut staging area and cpio manifest
*   Abandoned, primarily due to continued lack of upstream feedback


<!-- .slide: data-state="normal" id="reflinks-impl2" data-timing="20s" data-menu-title="Reflinks Implementation 2" -->
# Archiver Implementation: dracut-cpio

*   From-scratch cpio archiver written in Rust
*   Why?
    *   Need only support simple cpio `newc` format for kernel
    *   Rust standard library attempts reflink via `io::copy()`
    *   Upstream Dracut already evaluating Rust for other purposes
*   Integrated support for data segment padding


<!-- .slide: data-state="normal" id="reflinks-benchmarks" data-timing="20s" data-menu-title="Reflinks Benchmarking" -->
# Benchmarking

*   Tumbleweed 20210924 (5.14.6 kernel)
*   VM with 2 vCPUs and 8GiB RAM
*   Raw QEMU disk images stored in memory on hypervisor `tmpfs`
*   SUSE Dracut version 055, patched with dracut-cpio reflink support
    * **`xz`:** GNU cpio, `xz -0 --check=crc32 --memlimit-compress=50%`, `strip` symbols
    * **`zstd`:** GNU cpio, `zstd -3 -T0`, `strip` symbols
    * **Dracut reflink**: dracut-cpio enabled via `enhanced_cpio=yes`, compression and  symbol stripping disabled. Reflink friendly Dracut staging area (`tmpdir=/boot`).


<!-- .slide: data-state="normal" id="reflinks-perf-btrfs" data-timing="20s" data-menu-title="Reflinks Performance on Btrfs" -->
# Btrfs Performance

|   **Btrfs**                      |   Dracut `xz`  |    Dracut `zstd`   |    Dracut reflink   |
|:--------------------------------:|:--------------:|:------------------:|:-------------------:|
| Dracut runtime                   |  16.809s (ref) |  12.935s (-23.05%) |   10.292s (-38.59%) |
| QEMU boot time: kernel to /init  |   2361ms (ref) |  951.0ms (-59.71%) |   857.8ms (-63.66%) |
| **initramfs `fiemap` data**      |                |                    |                     |
|                      total bytes | 14647296 (ref) | 16191488 (+10.54%) |  34447360 (+135.2%) |
|            shared (deduplicated) |        0 (ref) |        0 (+-0.00%) | 31309824 (**note**) |
|                        exclusive | 14647296 (ref) | 16191488 (+10.54%) |  3137536 (**note**) |

**note**: Btrfs `fiemap` results varied significantly across multiple runs


<!-- .slide: data-state="normal" id="reflinks-perf-xfs" data-timing="20s" data-menu-title="Reflinks Performance on XFS" -->
# XFS Performance

|   **XFS**                        |    Dracut `xz` |    Dracut `zstd`   |   Dracut reflink   |
|:--------------------------------:|:--------------:|:------------------:|:------------------:|
| Dracut runtime                   |  14.180s (ref) |  10.437s (-26.40%) |  9.7687s (-31.11%) |
| QEMU boot time: kernel to /init  |   2373ms (ref) |  950.9ms (-59.92%) |  859.5ms (-63.78%) |
| **initramfs `fiemap` data**      |                |                    |                    |
|                      total bytes | 14675968 (ref) | 16158720 (+10.10%) | 34689024 (+136.4%) |
|            shared (deduplicated) |        0 (ref) |         0 (±0.00%) |           26525696 |
|                        exclusive | 14675968 (ref) | 16158720 (+10.10%) |  8163328 (-44.37%) |


<!-- .slide: data-state="normal" id="recommendations" data-timing="20s" data-menu-title="Recommendations" -->
# Recommendations

Initramfs images:

*   Reflink cpio archives, provided Btrfs or XFS requirements are met
*   zstd compress archives if reflinks aren't available
*   Disable initramfs modification time preservation

Kernel package updates:

*   Switch module compression to zstd
*   Enable "incremental" mode for depmod
*   Avoid duplicate initramfs rebuilds
