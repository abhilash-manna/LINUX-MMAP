<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" class="logo" width="120"/>

# but where my doubt arise from i see some where using /dev/mem for mmap and statisn that if the requested memory address is not present in page table it won't work.

can you clear this doubt

When you do

```c
int fd = open("/dev/mem", O_RDWR|O_SYNC);
void *p = mmap(0, size,
               PROT_READ|PROT_WRITE,
               MAP_SHARED, fd, phys_addr);
```

you are *not* using the generic “anonymous‐mapping” path in mm/mmap.c. Instead you are invoking the **/dev/mem** driver’s own mmap handler, which lives in drivers/char/mem.c:

```c
static int mem_mmap(struct file *filp, struct vm_area_struct *vma)
{
    unsigned long pfn = vma->vm_pgoff;
    size_t size     = vma->vm_end - vma->vm_start;

    /* Reject PFNs outside physical RAM or I/O ranges */
    if (!mem_is_allowed(pfn, size >> PAGE_SHIFT))
        return -EINVAL;

    /* Create PTEs mapping [vma->vm_start, vma->vm_end)
     * to physical PFNs [pfn, pfn + npages)
     */
    return remap_pfn_range(vma,
                           vma->vm_start,
                           pfn,
                           size,
                           vma->vm_page_prot);
}
```

Key points:

- **This is driver code,** not the generic anon‐VMA logic.
- When you call `mmap(/dev/mem)`, the kernel sets up **page-table entries (PTEs)** right then via `remap_pfn_range()`. Nothing must already be in your page tables—the driver installs the PTEs for you.
- The only reason such an mmap can “fail because page isn’t present” is that the mem driver first checks `mem_is_allowed(pfn, npages)`, which verifies that the requested PFN range is within allowed physical RAM or I/O windows. If you ask for an address outside those ranges, it refuses (`-EINVAL`).

Contrast with **anonymous mappings** (`MAP_ANONYMOUS`):
– The VM core creates a VMA with no file and no driver involved.
– No PTEs exist initially—on the very first access to each page, the generic anon‐fault path allocates a zeroed page and installs its PTE.

With **/dev/mem** you’re bypassing all of that: the *mem* driver’s `mmap()` immediately populates your page tables with mappings to *physical* addresses (via PFNs). If you try to map a PFN that the mem driver doesn’t recognize as valid RAM or I/O, it simply refuses up front—hence the “it won’t work if the page isn’t in the page table” misconception. In reality it *never* uses your existing page table; it builds new entries for you—but only for PFNs the driver has been told (by platform code) are legitimate to expose.

<div style="text-align: center">⁂</div>

[^1]: http://arxiv.org/pdf/1910.05398.pdf

[^2]: http://arxiv.org/pdf/2310.09124.pdf

[^3]: https://arxiv.org/pdf/2103.10779.pdf

[^4]: https://arxiv.org/html/2502.14220v1

[^5]: https://arxiv.org/pdf/2502.10923.pdf

[^6]: https://static-content.springer.com/esm/art:10.1186/2193-1801-3-494/MediaObjects/40064_2014_1199_MOESM1_ESM.pdf

[^7]: https://arxiv.org/abs/2401.15558

[^8]: https://arxiv.org/pdf/2307.10618.pdf

[^9]: https://dl.acm.org/doi/pdf/10.1145/3609308.3625267

[^10]: http://arxiv.org/pdf/2503.15065.pdf

[^11]: https://arxiv.org/pdf/2401.13154.pdf

[^12]: https://arxiv.org/html/2503.17685v1

[^13]: http://arxiv.org/pdf/2104.07699.pdf

[^14]: https://arxiv.org/pdf/2112.14013.pdf

[^15]: https://arxiv.org/pdf/2409.10946.pdf

[^16]: https://arxiv.org/pdf/1911.08367.pdf

[^17]: http://arxiv.org/pdf/2003.03304.pdf

[^18]: https://dl.acm.org/doi/pdf/10.1145/3565026

[^19]: https://arxiv.org/html/2409.11220v1

[^20]: http://arxiv.org/pdf/2403.04539.pdf

[^21]: https://stackoverflow.com/questions/27633439/why-is-mmio-space-mapped-to-dev-mem-as-a-single-pte-so-there-is-no-struct-page

[^22]: https://docs.kernel.org/admin-guide/mm/pagemap.html

[^23]: https://blogs.oracle.com/linux/post/yes-virginia-there-is-page-table-sharing-in-linux

[^24]: https://linux-kernel-labs.github.io/refs/heads/master/labs/memory_mapping.html

[^25]: https://unix.stackexchange.com/questions/167948/how-does-mmaping-dev-mem-work-despite-being-from-unprivileged-mode

[^26]: https://www.ibm.com/docs/ssw_aix_71/com.ibm.aix.files/mem.htm

[^27]: https://docs.kernel.org/arch/x86/pat.html

[^28]: https://man7.org/linux/man-pages/man2/mmap.2.html

[^29]: https://adaptivesupport.amd.com/s/question/0D52E00006hpguVSAQ/ultrascale-fails-to-correctly-mmap-devmem?language=en_US

[^30]: https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842412/Accessing+BRAM+In+Linux

[^31]: https://www.xml.com/ldd/chapter/book/ch13.html

