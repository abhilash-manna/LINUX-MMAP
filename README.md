# LINUX-MMAP
How and Why - Mmap in Linux

# make a diagram of mmap usage from user space to kernel.

The diagram below illustrates the complete `mmap` call flow, from the user‐space request through the kernel’s syscall layer, into the driver’s `.mmap` handler, and finally updating the process’s page tables via `remap_pfn_range` or similar helpers:

![Flowchart illustrating the mmap call flow from user space to kernel driver and page tables.]

![Flowchart illustrating the mmap call flow from user space to kernel driver and page tables.](https://user-gen-media-assets.s3.amazonaws.com/gpt4o_images/24d423b9-833c-45e2-8024-fc86d266a409.png)

Flowchart illustrating the mmap call flow from user space to kernel driver and page tables.


# diagram shows how  memory is mapped to process address space

Below is a detailed diagram showing how device or register memory is mapped into a process’s virtual address space:

![Diagram of memory mapping into process virtual address space via mmap]

![Diagram of memory mapping into process virtual address space via mmap](https://user-gen-media-assets.s3.amazonaws.com/gpt4o_images/91109781-b360-4968-ad65-fdce962620d5.png)

Diagram of memory mapping into process virtual address space via mmap
