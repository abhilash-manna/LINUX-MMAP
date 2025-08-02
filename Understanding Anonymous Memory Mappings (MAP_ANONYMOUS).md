# Understanding Anonymous Memory Mappings (MAP_ANONYMOUS) and Process Fork Sharing

When you call

```c
mmap(NULL, length,
     PROT_READ|PROT_WRITE,
     MAP_SHARED|MAP_ANONYMOUS,
     -1, 0);
```

you are creating a region of memory in your process’s virtual address space that:

- **Is not backed by any file**: There is no file descriptor involved. The kernel simply allocates pages of RAM (or swap) to back this region.
- **Lives entirely in RAM (or swap)**: Since there’s no file, the contents begin as zero-filled pages.
- **Can be shared**: The `MAP_SHARED` flag means that if multiple processes map the *same* anonymous region, writes by one process become visible to the others.


## How Parent and Child Share Anonymous Memory

1. **Anonymous mapping in parent**
When the parent calls `mmap(... MAP_SHARED|MAP_ANONYMOUS ...)`, the kernel:
    - Allocates an anonymous memory region of the requested size.
    - Establishes page table entries mapping the parent’s virtual addresses to these new physical pages.
2. **Forking the process**
On `fork()`, Linux duplicates the parent’s page tables for the child. For **shared** mappings:
    - Both parent and child page tables point to the *same* physical pages.
    - The pages are marked copy-on-write only if the mapping is private; for `MAP_SHARED`, writes go straight to the original pages.
3. **Reading and writing**
    - Reads by parent or child see the same data because they’re accessing identical physical pages.
    - Writes by either process update those pages directly, so the other process immediately sees the change.

### Step-by-Step Execution

```c
int *p = mmap(NULL, sizeof *p,
              PROT_READ|PROT_WRITE,
              MAP_SHARED|MAP_ANONYMOUS,
              -1, 0);
*p = 42;           // Kernel allocates pages and stores 42

pid_t pid = fork();
if (pid == 0) {
    // Child process
    printf("Child sees %d\n", *p);  // Prints 42
    *p = 100;                       // Writes to the same page
} else {
    sleep(1);
    printf("Parent sees %d\n", *p); // Prints 100, updated by child
}
```

1. **`mmap()` in the parent**:
    - Kernel zero-fills a page and maps it at address `p` in the parent.
    - Parent writes `42` into that page.
2. **`fork()`**:
    - Child’s page table entries for that region are a direct copy of the parent’s.
    - Because the mapping is `MAP_SHARED`, both tables refer to the *same* physical page.
3. **Child writes**:
    - `*p = 100;` updates the shared page.
4. **Parent reads**:
    - After `sleep()`, parent reads `*p` and sees `100`, because it’s the same physical memory.

## Why “Not Backed by Any File”?

- **File-backed mapping** uses a file descriptor: changes can be synchronized back to a file on disk.
- **Anonymous mapping** uses no file; it is entirely in memory (or swap).
- Think of it as the kernel providing you a private heap page (like `malloc`), but with the option for *multiple processes* to share that heap by using `MAP_SHARED`.


### Key Takeaways

- `MAP_ANONYMOUS` tells the kernel “no file—just give me zero-filled memory pages.”
- `MAP_SHARED` tells the kernel “I want changes to these pages to be visible to any other process that maps the same region.”
- `fork()` duplicates address spaces so that shared mappings remain truly shared, not copied.

This mechanism is often used for **inter-process communication (IPC)** without requiring files, pipes, or sockets—just shared pages in memory.

