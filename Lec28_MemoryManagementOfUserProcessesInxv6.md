**Memory management of user processes**
- User process needs memory pages to build its address space
    * User part of memory image (user code/data/stack/heap)
    * Page table (mappings to user memory image, as well as to kernel code/data)
- Free list of kernel used to allocate memory for user processes via kalloc()
- New virtual address space for a process is created during:
    * init process creation
    * fork system call
    * exec system call
- Existing virtual address space modified in sbrk system call (expand heap)
- How is page table of a process constructed?
    * Start with one page for the outer page directory
    * Allocate inner page tables on demand (if no entries present in inner page table, no need to allocate a page for it) as memory image created or updated

**Functions to build page table**
- Every page table begins with setting up kernel mappings (mappings to OS code and data) in setupkvm()
- Outer pgdir allocated
- Kernel mappings defined in "kmap" added to page table by calling "mappages"
- After setupkvm(), user page table mappings added
- Page table entries added by "mappages"
    * Arguments: page directory, range of virtual addresses, physical addresses to map to, permissions of the pages
    * For each page, walks page table, get pointer to PTE via function "walkpgdir", fills it with physical addresses and permissions
- Function "walkpgdir" walks page table, returns PTE of a virtual address
    * Can allocate inner page table if it doesn't exist

**Fork: copying memory image**
- Function "copyuvm" called by parent to copy parent memory image to child
    * <u>Create new page table for child</u>
    * <u>Walk through parent memory image page by page and copy it to child, while adding child page table mappings</u>
- For each page in parent
    * fetch PTE, get physical address, permissions
    * Allocate new page for child, and copy contents of parent's page to new page of child
    * Add a PTE from virtual address to physical address of new page in child page table
- Real operating systems do copy-on-write: child page table also points to parent pages until either of them modifies it
    * Here, xv6 creates separate memory images for parent and child right away

**Growing memory image: sbrk**
- Initially heap is empty, program "break" (end of user memory) is at end of stack
    * sbrk() system call invoked by malloc to expand heap
- To grow memory, allocuvm allocates new pages, adds mappings into page table for new pages
- Whenever page table updated, must update cr3 register and TLB (done even during context switching)

**allocuvm: grow address space**
- Walk through new virtual addresses to be added in page size chunks
- Allocate new page, add it to page table with suitable user permissions
    * If you don't add mapping to the page table, you won't be able to access the page
- Similarly deallocuvm shrinks memory iamge, frees up page

**Exec system call**
- Read ELF binary file from disk to memory
- Start with new page table, add mappings to new executable pages and grow virtual address space
    * Do not overwrite old page table yet
- After executable is copied to memory image, allocate 2 pages for stack (one is guard page, permissions cleared(no one can access it), access will trap)
- Push exec arguments onto user stack for main functions of user program
    * Stack has return address, argc, argv array (pointers to variable sized arguments), and the arguments themselves
- If no errors so far, switch to new page table that is pointing to new memory image
    * If any error, go back to old memory image (exec returns with error)
- Set eip to trapframe to start at entry point of new program
    * Returning from trap, processes will run new executable

**Summary**
- Memory management for user processes
    * Build page table: start with kernel mappings, add user entries to build virtual address space
    * Memory management code in fork, exec, sbrk