# Mapping Device or Register Memory into User Space with mmap in Linux Drivers

**Key Takeaway:** To expose device or register memory to user space, implement the driver’s `.mmap` file operation by calling `remap_pfn_range()` (or `io_remap_pfn_range()` for I/O BARs). Validate offsets and lengths, set appropriate page protections, and supply different implementations for character, platform, and PCI drivers.

## 1. Character Device Driver Example

A minimal char driver allocates a kernel buffer and remaps it into user space on `mmap`:

```c
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/mm.h>
#include <linux/slab.h>

#define DEVICE_NAME "chardev"
#define BUFFER_SIZE 0x1000

static char *kernel_buffer;
static int major;

static int dev_open(struct inode *inode, struct file *file)
{
    kernel_buffer = kmalloc(BUFFER_SIZE, GFP_KERNEL);
    if (!kernel_buffer) return -ENOMEM;
    file->private_data = kernel_buffer;
    return 0;
}

static int dev_release(struct inode *inode, struct file *file)
{
    kfree(file->private_data);
    return 0;
}

static int dev_mmap(struct file *file, struct vm_area_struct *vma)
{
    unsigned long pfn = virt_to_phys(file->private_data) >> PAGE_SHIFT;
    size_t size = vma->vm_end - vma->vm_start;

    if (size > BUFFER_SIZE)
        return -EINVAL;

    if (remap_pfn_range(vma,
                        vma->vm_start,
                        pfn + vma->vm_pgoff,
                        size,
                        vma->vm_page_prot))
        return -EAGAIN;

    return 0;
}

static const struct file_operations fops = {
    .owner   = THIS_MODULE,
    .open    = dev_open,
    .release = dev_release,
    .mmap    = dev_mmap,
};

static int __init chardev_init(void)
{
    major = register_chrdev(0, DEVICE_NAME, &fops);
    return (major < 0) ? major : 0;
}

static void __exit chardev_exit(void)
{
    unregister_chrdev(major, DEVICE_NAME);
}

module_init(chardev_init);
module_exit(chardev_exit);
MODULE_LICENSE("GPL");
```

**Explanation:**

- Allocate a buffer on open and store its pointer in `file->private_data`.
- In `.mmap`, compute the physical page frame number (PFN) with `virt_to_phys()`, validate the size, then call `remap_pfn_range()` to map it into user space [^1].


## 2. Platform Driver Example

For memory‐mapped hardware described in Device Tree, use `devm_of_iomap()` and remap via a simple `.mmap` handler:

```c
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/of.h>
#include <linux/of_device.h>
#include <linux/io.h>
#include <linux/mm.h>

struct plat_dev {
    void __iomem *regs;
};

static int plat_probe(struct platform_device *pdev)
{
    struct plat_dev *pd;
    pd = devm_kzalloc(&pdev->dev, sizeof(*pd), GFP_KERNEL);
    pd->regs = devm_of_iomap(&pdev->dev, pdev->dev.of_node, 0, NULL);
    platform_set_drvdata(pdev, pd);
    return 0;
}

static int plat_mmap(struct file *file, struct vm_area_struct *vma)
{
    struct plat_dev *pd = file->private_data;
    unsigned long phys = virt_to_phys(pd->regs) >> PAGE_SHIFT;
    unsigned long size = vma->vm_end - vma->vm_start;

    if (size > resource_size(pdev->resource))
        return -EINVAL;

    return remap_pfn_range(vma,
                           vma->vm_start,
                           phys + vma->vm_pgoff,
                           size,
                           vma->vm_page_prot);
}

static const struct of_device_id of_match[] = {
    { .compatible = "vendor,device" },
    { }
};

static const struct file_operations plat_fops = {
    .owner = THIS_MODULE,
    .mmap  = plat_mmap,
};

static struct miscdevice plat_misc = {
    .minor = MISC_DYNAMIC_MINOR,
    .name  = "platdev",
    .fops  = &plat_fops,
};

static struct platform_driver plat_driver = {
    .probe = plat_probe,
    .driver = {
        .name = "platdrv",
        .of_match_table = of_match,
    },
};

module_platform_driver(plat_driver);
module_misc_device(plat_misc);
MODULE_LICENSE("GPL");
```

**Explanation:**

- `devm_of_iomap()` maps the device’s register range into kernel virtual space.
- In `.mmap`, derive the PFN from that mapping and call `remap_pfn_range()` [^2].


## 3. PCI Driver Example

For PCI BAR regions, use the helper `pci_mmap_resource_range()` and `io_remap_pfn_range()`:

```c
#include <linux/pci.h>
#include <linux/mm.h>

static int pci_mmap(struct file *file, struct vm_area_struct *vma)
{
    struct pci_dev *pdev = file->private_data;
    int bar = 0; /* Use BAR0 */
    return pci_mmap_resource_range(pdev,
                                   bar,
                                   vma,
                                   PCI_MMAP_MEM,
                                   0 /* no write-combine */);
}

static const struct file_operations pci_fops = {
    .owner = THIS_MODULE,
    .mmap  = pci_mmap,
};

static int pci_probe(struct pci_dev *pdev,
                     const struct pci_device_id *id)
{
    pci_enable_device(pdev);
    pci_set_drvdata(pdev, pdev);
    /* register char device or misc device with pci_fops */
    return 0;
}
```

**Explanation:**

- `pci_mmap_resource_range()` checks bounds, sets `vma->vm_page_prot` appropriately, and calls `io_remap_pfn_range()` to map the BAR into user space [^3].


## 4. Using `vm_operations_struct` for On‐Demand Mapping

If only some pages are accessed, defer mapping via a fault handler:

```c
static const struct vm_operations_struct simple_vm_ops = {
    .fault = simple_fault,
};

static int simple_mmap(struct file *file, struct vm_area_struct *vma)
{
    vma->vm_ops = &simple_vm_ops;
    vma->vm_flags |= VM_RESERVED;
    vma->vm_private_data = file->private_data;
    simple_open(vma);
    return 0;
}

static vm_fault_t simple_fault(struct vm_fault *vmf)
{
    struct page *page = virt_to_page(vmf->vma->vm_private_data);
    get_page(page);
    vmf->page = page;
    return VM_FAULT_NOPAGE;
}
```

**Explanation:**

- Pages are mapped on first access by supplying `.fault` in `vm_operations_struct` [^4].


## User‐Space Access

In user space, open the device and call `mmap()`:

```c
int fd = open("/dev/chardev", O_RDWR);
void *addr = mmap(NULL, 0x1000,
                  PROT_READ | PROT_WRITE,
                  MAP_SHARED, fd, 0);
if (addr == MAP_FAILED) {
    perror("mmap");
    return 1;
}
/* Access registers or buffer at addr */
```

**Summary of Driver Types and `mmap` Usage**


| Driver Type | Kernel Call | Use Case | Page Fault vs Immediate Mapping |
| :-- | :-- | :-- | :-- |
| Character Device | remap_pfn_range | Kernel buffer or custom memory | Immediate |
| Platform Driver | remap_pfn_range (after devm_of_iomap) | MMIO from Device Tree | Immediate |
| PCI Driver | pci_mmap_resource_range → io_remap_pfn_range | PCI BAR memory | Immediate |
| On-Demand Mapping | vm_ops `.fault` + remap inside fault handler | Sparse or large mappings | Deferred |

**References:**
[^1] remap_pfn_range usage in simple char driver [^1]
[^2] devm_of_iomap and remap_pfn_range in platform drivers [^5]
[^3] pci_mmap_resource_range implementation [^6]
[^4] Fault‐based mmap via `vm_fault` handler [^7]

<div style="text-align: center">⁂</div>

[^1]: https://labs.withsecure.com/content/dam/labs/docs/mwri-mmap-exploitation-whitepaper-2017-09-18.pdf

[^2]: https://www.xml.com/ldd/chapter/book/ch13.html

[^3]: https://www.youtube.com/watch?v=cIUvsfutz-U

[^4]: https://linux-mm.org/DeviceDriverMmap

[^5]: https://stackoverflow.com/questions/72279627/linux-device-drivers-mapping-mmio-with-devm-of-iomap-not-working

[^6]: https://android.googlesource.com/kernel/common/+/a4eacf3227bd/drivers/pci/mmap.c

[^7]: https://stackoverflow.com/questions/30069306/driver-mmap-operation-page-table-creation

[^8]: https://docs.oracle.com/cd/E19120-01/open.solaris/819-3196/devmap-24338/index.html

[^9]: https://stackoverflow.com/questions/8788289/how-remap-pfn-range-remaps-kernel-memory-to-user-space

[^10]: https://discuss.python.org/t/mmap-char-device-driver/51930

[^11]: https://stackoverflow.com/questions/66893486/how-can-my-pci-device-driver-remap-pci-memory-to-userspace

[^12]: https://linux-kernel-labs.github.io/refs/pull/183/merge/labs/memory_mapping.html

[^13]: https://github.com/torvalds/linux/blob/master/drivers/pci/mmap.c

[^14]: https://www.marcusfolkesson.se/blog/mmap-memory-between-kernel-and-userspace/

[^15]: https://github.com/tatetian/linux-driver-examples/blob/master/scullp/mmap.c

[^16]: https://www.kernel.org/doc/html/next/PCI/pci.html

[^17]: https://gist.github.com/laoar/4a7110dcd65dbf2aefb3231146458b39

[^18]: https://stackoverflow.com/questions/21723594/how-to-implement-memory-map-feature-in-device-drivers-in-linux

[^19]: https://unix.stackexchange.com/questions/643077/how-can-my-pci-device-driver-remap-pci-memory-to-userspace

[^20]: https://www.kernel.org/doc/html/v5.9/core-api/bus-virt-phys-mapping.html

[^21]: https://github.com/paraka/mmap-kernel-transfer-data/blob/master/mmap-example.c

[^22]: https://stackoverflow.com/questions/12790382/mapping-memory-reserved-by-mmap-kernel-boot-param-into-user-space

[^23]: https://stackoverflow.com/questions/71394918/how-to-mmap-memory-for-miscdevice-why-my-drivers-mmap-is-not-called

[^24]: https://community.nxp.com/t5/-/-/m-p/207393

[^25]: https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18841683/Linux+Reserved+Memory里

[^26]: https://groups.google.com/g/linux.kernel/c/9JD4qyriGds

[^27]: https://kernel.googlesource.com/pub/scm/linux/kernel/git/bcousson/linux-omap-dt/+/for_3.13/dts/drivers/of/of_reserved_mem.c

[^28]: https://www.sobyte.net/post/2022-03/mmap/

[^29]: https://stackoverflow.com/questions/15610570/what-is-the-difference-between-a-linux-platform-driver-and-normal-device-driver

[^30]: https://github.com/Xilinx/linux-xlnx/blob/master/drivers/of/of_reserved_mem.c

[^31]: https://e2e.ti.com/support/processors-group/processors/f/processors-forum/6135/linux-memory-mapped-drivers-and-nor-flash

[^32]: https://github.com/torvalds/linux/blob/master/drivers/of/of_reserved_mem.c

[^33]: https://embetronicx.com/tutorials/linux/device-drivers/device-file-creation-for-character-drivers/

[^34]: https://coral.googlesource.com/linux-imx/+/refs/tags/7-3/drivers/of/of_reserved_mem.c

[^35]: https://subscription.packtpub.com/book/cloud-and-networking/9781801079518/2/ch02lvl1sec08/understanding-the-connection-between-the-process-the-driver-and-the-kernel

[^36]: https://codebrowser.dev/linux/linux/drivers/of/of_reserved_mem.c.html

