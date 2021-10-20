<!-- .slide: data-state="divider" id="divider" data-timing="20s" data-menu-title="Divider with image" -->
# Initramfs Reflinks

<a title="By Brandenads, Public Domain" href="https://en.wikipedia.org/wiki/Tetris#/media/File:Typical_Tetris_Game.svg">
    <img alt="Tetris" src="images/Typical_Tetris_Game_wik_public_domain_brandenads.svg"/>
</a>


<!-- .slide: data-state="normal" id="reflinks-intro" data-timing="20s" data-menu-title="Reflinks Introduction" -->
# Initramfs Reflinks

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
    *   **See Ludwig's talk: usrmerge and beyond**
*   Causes significant fragmentation


<!-- .slide: data-state="normal" id="reflinks-boot" data-timing="20s" data-menu-title="Reflinks Boot" -->
# Reflinks: Boot Time Considerations

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

|   **Btrfs**                      | Dracut `xz` | Dracut `zstd` | Dracut reflink |
|:--------------------------------:|:-----------:|:-------------:|:--------------:|
| Dracut runtime                   |         17s |           13s |            10s |
| QEMU boot time: kernel to /init  |        2.4s |          1.0s |           0.9s |
| **initramfs `fiemap` data**      |             |               |                |
|                       total size |         14M |           15M |            33M |
|            shared (deduplicated) |           - |             - |            30M |
|                        exclusive |         14M |           15M |             3M |

**Note**: Btrfs `fiemap` results varied significantly across multiple runs


<!-- .slide: data-state="normal" id="reflinks-perf-xfs" data-timing="20s" data-menu-title="Reflinks Performance on XFS" -->
# XFS Performance

|   **XFS**                        | Dracut `xz` | Dracut `zstd` | Dracut reflink |
|:--------------------------------:|:-----------:|:-------------:|:--------------:|
| Dracut runtime                   |         14s |           10s |            10s |
| QEMU boot time: kernel to /init  |        2.4s |          1.0s |           0.9s |
| **initramfs `fiemap` data**      |             |               |                |
|                       total size |         14M |           15M |            33M |
|            shared (deduplicated) |           - |             - |            25M |
|                        exclusive |         14M |           15M |             8M |


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
