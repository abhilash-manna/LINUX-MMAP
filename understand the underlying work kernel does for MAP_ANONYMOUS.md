<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" class="logo" width="120"/>

# i want to understand the underlying work kernel does for MAP_ANONYMOUS.

There  must be a driver which takes care the mapping.
we write explicit mmap implementation for driver code to use mmap from user space for some device driver, how this is different.

so my doubt is how and which implementation of code works. is it a generic mmap logic implemented in the driver side to support MAP_ANONYMOUS

The kernel implements anonymous mappings entirely in its generic mm subsystem—no “MAP_ANONYMOUS driver” is needed or involved. When a user calls

```c
mmap(NULL, length,
     PROT_READ|PROT_WRITE,
     MAP_SHARED|MAP_ANONYMOUS,
     -1, 0);
```

the following happens under the hood:

1. A new `vm_area_struct` (VMA) is created and inserted into the process’s `mm_struct` VMA tree.
2. Because the flags include `MAP_ANONYMOUS`, the VMA’s file pointer is set to NULL and an **anon_vma** object is allocated, linking this VMA into the anonymous‐mapping reverse‐mapping (rmap) trees[^1][^2].
3. Physical pages are **not** allocated immediately; instead pages are faulted in on demand by the standard page‐fault handler. When the process first reads or writes any address in that VMA:
    - The MMU traps, the core’s page‐fault path runs (in `do_page_fault()`).
    - The generic **anon fault handler** (`__handle_mm_fault()`) sees there is no file and calls into the anon‐VMA fault logic in mm/mmap.c.
    - That logic allocates a zeroed page (`alloc_page()`), wires it into the page tables, and links it into the anon_vma’s rmap[^1].

All of this is pure VM infrastructure in **mm/mmap.c**, **mm/rmap.c**, and related files—device drivers never write any of it.

— **Contrast with driver mmap( ) implementations**
When you write a character or uio driver that supports `mmap()`, you fill in a `struct file_operations` entry:

```c
static const struct file_operations fops = {
    .open  = my_open,
    .mmap  = my_mmap,
    /* … */
};
```

Your `my_mmap(struct file *filp, struct vm_area_struct *vma)` typically does one of two things:

- **remap_pfn_range( )**—for a contiguous kernel buffer (e.g. allocated by `__get_free_pages()`), it takes the physical‐page frame numbers and maps them into the user’s VMA in one call.
- **vm_insert_page( )** or a custom `vm_ops->fault` handler—if your buffer is noncontiguous (e.g. `vmalloc()` or DMA memory), you either map page by page or let your own fault callback allocate pages on demand.

This driver‐supplied `mmap` is only invoked for file‐backed mappings on your device’s special‐file; it is entirely separate from the generic anonymous‐mapping logic in mm/.

**Key Differences**

- **Mapper:**
– *Anonymous:* Handled by the generic `do_mmap()` → `anon_vma` → `handle_mm_fault()` code in the VM core.
– *Driver:* Handled by your `fops->mmap` callback and any `vm_ops` you attach.
- **Backing store:**
– *Anonymous:* No file, zero‐filled pages, paging and swap managed by VM core.
– *Driver:* Backed by your driver’s buffer (physically contiguous or not), you decide if and how pages are allocated, wired, or remapped.
- **Faulting:**
– *Anonymous:* Generic anon‐fault path allocates pages on first access.
– *Driver:* Either no faults (pre‐populated via `remap_pfn_range`) or custom fault callback (`vm_ops->fault`) to supply your buffer’s pages.

There is **no “generic mmap driver”** for `MAP_ANONYMOUS`—the kernel’s VM core simply treats anonymous mappings as a special case of VMA with no file, tracked by `anon_vma` structures and serviced by its built‐in fault handler. Your own driver’s `mmap` is only involved when you call it via `open(“/dev/…”); mmap(…)` on a device file.

<div style="text-align: center">⁂</div>

[^1]: https://stackoverflow.com/questions/62094006/which-file-is-an-anonymous-memory-mapping-mapped-to-in-linux

[^2]: https://ieeexplore.ieee.org/document/10974794/

[^3]: https://arxiv.org/abs/2409.10946

[^4]: http://link.springer.com/10.1007/s10586-013-0309-0

[^5]: https://www.semanticscholar.org/paper/f224ffae0cbbe02eff796e9c222eaa7b8719c689

[^6]: http://ieeexplore.ieee.org/document/4696190/

[^7]: http://ieeexplore.ieee.org/document/4673691/

[^8]: http://ieeexplore.ieee.org/document/4534808/

[^9]: https://ieeexplore.ieee.org/document/6495881/

[^10]: http://link.springer.com/10.1007/3-540-48228-8_26

[^11]: https://arxiv.org/html/2412.01059v1

[^12]: https://dl.acm.org/doi/pdf/10.1145/3605098.3635951

[^13]: http://ispras.ru/proceedings/docs/2018/30/3/isp_30_2018_3_121.pdf

[^14]: https://dl.acm.org/doi/pdf/10.1145/3609308.3625267

[^15]: https://arxiv.org/pdf/2409.10946.pdf

[^16]: https://static-content.springer.com/esm/art:10.1186/2193-1801-3-494/MediaObjects/40064_2014_1199_MOESM1_ESM.pdf

[^17]: https://arxiv.org/pdf/2310.00933.pdf

[^18]: https://dl.acm.org/doi/pdf/10.1145/3576915.3623206

[^19]: http://arxiv.org/pdf/2307.14471.pdf

[^20]: https://arxiv.org/pdf/2411.18094.pdf

[^21]: https://arxiv.org/pdf/2212.12671.pdf

[^22]: https://groups.google.com/g/comp.os.linux.development.system/c/M5pY_hxG5kA

[^23]: https://android.googlesource.com/kernel/mediatek/+/android-mtk-3.18/include/linux/rmap.h?autodive=0%2F%2F%2F

[^24]: https://lwn.net/Articles/1015762/

[^25]: https://www.baeldung.com/linux/memory-mapped-file-vs-anonymous-memory

[^26]: https://access.redhat.com/solutions/1228953

[^27]: https://www.slideshare.net/slideshow/memory-mapping-implementation-mmap-in-linux-kernel-251745285/251745285

[^28]: https://codebrowser.dev/linux/linux/mm/rmap.c.html

[^29]: https://pubs.opengroup.org/onlinepubs/009695399/functions/mmap.html

[^30]: https://docs.kernel.org/mm/process_addrs.html

[^31]: https://man7.org/linux/man-pages/man2/mmap.2.html

[^32]: https://nvd.nist.gov/vuln/detail/CVE-2023-52935

[^33]: https://news.ycombinator.com/item?id=22107066

[^34]: https://docs.huihoo.com/doxygen/linux/kernel/3.7/structanon__vma.html

[^35]: https://www.postgresql.org/docs/current/kernel-resources.html

[^36]: https://arxiv.org/html/2409.10946v1

[^37]: https://news.ycombinator.com/item?id=19805675

