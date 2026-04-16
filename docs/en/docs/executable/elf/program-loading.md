# Program Loading

The program loading process is essentially the system creating or expanding a process image. It simply copies segments from the file to virtual memory segments according to certain rules. A process only requests corresponding physical pages when it actually uses the logical pages during execution. Typically, many pages in a process are never referenced. Therefore, delaying physical reads and writes can improve system performance. To achieve this efficiency, the file offsets and virtual addresses of segments in executable files and shared object files must be appropriate, meaning they must be integer multiples of the page size.

On the Intel architecture, virtual addresses and file offsets must be integer multiples of 4KB (or a larger power of 2).

Below is an example of the memory layout when an executable file is loaded into memory:

![](./figure/executable_file_example.png)

The corresponding interpretations of the code segment and data segment are as follows:

![](./figure/program_header_segments.png)

In this example, although the code segment and data segment are equal modulo 4KB, there are still at most four pages containing impure code or data. Of course, in practice this depends on the page size or the file system block size.

- The first page of the code segment contains the ELF header, the program header table, and other information.
- The last page of the code segment contains a copy of the beginning of the data segment.
- The last page of the data segment contains a copy of the end of the code segment. The exact amount is not specified.
- The last part of the data segment may contain information unrelated to the program's execution.

Logically, the system enforces memory permissions as if each segment's permissions were completely independent; segment addresses are adjusted to ensure that each logical page in memory has only one set of permission types. In the example above, the last part of the file's code segment and the beginning of the data segment are both mapped twice: once at the data segment's virtual address and once at the code segment's virtual address.

The end of the data segment requires careful handling of uninitialized data. Generally, the system requires them to start with zeros. Therefore, if the last page of a file contains information not in the logical page, the remaining data must be initialized to zero. The impure data in the remaining three pages is not logically part of the process image, and the system may choose to delete them. The corresponding virtual memory image of this file is as follows (assuming each page is 4KB):

![](./figure/process_segments_image.png)

When loading segments, there are differences between executable files and shared object files. Executable files typically contain absolute code. For the program to execute correctly, each segment should be at the virtual address used to build the executable file. Therefore, the system directly uses p_vaddr as the virtual address.

On the other hand, shared object files typically contain position-independent code. This means that the same segment may have different virtual addresses in different processes, but this does not affect the program's execution behavior. Although the system selects different virtual addresses for different processes, it still maintains the relative addresses of segments. Because position-independent code uses relative addresses between different segments, the difference between virtual addresses in virtual memory must be the same as the difference between corresponding virtual addresses in the file. Below shows possible situations for different processes using the same shared object file, describing relative address addressing, and this table also shows the base address calculation method.

![](./figure/shared_object_segments_addresses.png)
