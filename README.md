Download Link: https://assignmentchef.com/product/solved-cs2200-project-3
<br>
<h1>1        Introduction</h1>

<strong>Read the entire document before starting. There are critical pieces of information and hints along the way.</strong>

In this project, you will be implementing a virtual memory system simulator. You have been given a simulator which is missing some critical parts. You will be responsible for implementing these parts. Detailed instructions are in the files to guide you along the way. If you are having trouble, we <strong>strongly suggest </strong>that you take the time to read about the material from the book and class notes.

<strong>There are 10 problems in the files that you will complete. The files that you will be changing are the following:</strong>

<ul>

 <li>h – Break down a virtual address into its components.</li>

 <li>c – Initialize any necessary bookkeeping and implement address translation.</li>

 <li>c – Implement the page fault handler.</li>

 <li>c – Write frame eviction and the LRU algorithm.</li>

 <li>c – Calculate the Average Access Time (AAT)</li>

</ul>

You will fill out the functions in these files, and then validate your output against the given outputs. If you are struggling with writing the code, then step back and review the concepts. Be sure to start early, ask Piazza questions, and visit us in office hours for extra help!

<h1>2        Page Splitting</h1>

In most modern operating systems, user programs access memory using virtual addresses. The hardware and the operating system work together to turn the virtual address into a physical address, which can then be used to address into physical memory. The first step of this process is to translate the virtual address into two parts: the higher order bits for the VPN, and the lower bits for the page offset.

In page_splitting.h, complete the vaddr_vpn and vaddr_offset functions. These will be used to split a virtual address into its corresponding page number and page offset. You will need to use the parameters for the virtual memory system defined in pagesim.h (PAGE_SIZE, MEM_SIZE, etc.).

<h1>3        Memory Organization</h1>

The simulator simulates a system with 1MB of physical memory. Throughout the simulator, you can access physical memory through the global variable uint8_t mem[] (an array of bytes called “mem”). You have access to, and will manage, the entirety of physical memory.

The system has a 24-bit virtual address space and memory is divided into 16KB pages.

Like a real computer, your page tables and data structures live in physical memory too! Both the page table and the frame table fit in a single page in memory, and you’ll be responsible for placing these structures into memory.

Note: Since user data and operating system structures such as the frame table and page tables coexist in the same physical memory, we must have some way to differentiate between the two, and keep user pages from taking over system pages.

Modern day operating systems often solve this problem by dividing physical memory up into a “kernel space” and a “user space”, where kernel space typically lies below a certain address and user space above. For this project, we’ll take a simpler approach: Every frame has a “protected” bit, which we’ll set to “1” for system frames and “0” for user frames.

<h1>4        Initialization</h1>

Before we can begin accessing pages, we will need to set up the frame table (sometimes known as a “reverse lookup table”). After that, for every process that starts, you’ll need to give it a page table.

For simplicity, we always place the frame table in physical frame 0. To set up the frame table, you need to initialize the pointer to the start of the frame table. Remember that this frame table belongs in a frame in memory – the first frame, so the pointer to the start of the frame table is just a frame table entry pointer. Don’t forget to mark this first frame as “protected”. We will <strong>never evict the frame table</strong>. To do this, we set a protected bit. During your page replacement, you will need to make sure that you never choose a protected frame as your victim.

Since processes can start and stop any time during your computer’s lifetime, we must be a little more sophisticated in choosing which frames to place their page tables in. For now, we won’t worry about the logistics of choosing a frame–just call the free_frame function you’ll write later in page_replacement.c. (Do we ever want to evict the frame containing the page table while the process is running?)

Your task is to fill out the following functions in paging.c:

<ol>

 <li>system_init()</li>

 <li>proc_init()</li>

</ol>

Each function listed above has helpful comments in the file. You may add any global variables or helper functions you deem necessary.

Each frame contains PAGE_SIZE bytes of data, therefore to access the start of the <em>i</em>-th frame in memory, you can use mem + (i * PAGE_SIZE).

<h1>5        Context Switches and the Page Table Base Register</h1>

As you know, every process has its own page table. When the processor needs to perform a page lookup, it must know which page table to look in. This is where the <em>page table base register </em>(PTBR) comes in.

In the simulator, you can access the page table base register through the global variable pfn_t PTBR.

Implement the context_switch function in paging.c. Your job is to update the PTBR to refer to the new process’s page table. This function will be very simple.

Going forward, pay close attention to the type of the PTBR. The PTBR holds a physical frame number (PFN), not a virtual address. Think about why this must be.

<h1>6        Reading and Writing Memory</h1>

The ability to allocate physical frames is useless if we cannot read or write to them. In this section, you will add functionality for reading and writing individual bytes to memory.

Because processes operate on a virtual memory space, it is necessary to first translate a virtual address supplied by a process into its corresponding physical address, which then will be used access the location in physical memory. This is accomplished using the page table, which contains all of a process’s mappings from virtual addresses to physical addresses.

Implement the mem_access function in paging.c. You will need to use the passed-in virtual address to find the correct page table entry and the offset within the corresponding page. <strong>HINT: </strong>Use the page splitting functions that you wrote earlier in the project.

Once you have identified the correct page table entry, you must use this to find the corresponding physical frame and its address in memory, and then perform the read or write at the proper location within the page. (Remember that the simulator’s memory is represented by the mem array).

Keep in mind that not all entries in a process’s page table have necessarily been mapped. Entries not yet mapped are marked as invalid, and an attempt to access an invalid address should generate a page fault. You will write the page_fault() function in the next section, so for now just assume that it has successfully allocated a page for that address after it returns.

Make sure to mark the containing page as “dirty” in the process’s page table on a write. These bits will be used later when deciding on what pages should be evicted first, and if an evicted page needs to be written to the disk to preserve its content.

<h1>7        Eviction and Replacement</h1>

Recall that when a CPU encounters an invalid VPN to PFN mapping in the page table, the OS allocates a new frame for the page by either finding an empty frame or evicting a page from a frame that is in use. In this section, you will be implementing a page fault and replacement mechanism.

Implement the function page_fault() in page_fault.c. A page fault occurs when the CPU attempts to translate a virtual address, but finds no valid entry in the page table for the VPN. To handle the page fault, you must find a frame to hold the page (call free_frame(), then update the page table and frame table to reference that frame.

Next, we will turn our attention to the eviction process in page_replacement.c.

If you ask the system for a free frame when all the frames are in use, the operating system must select an in-use frame and re-use it, “evicting” any existing page that was previously using the frame. Implement this logic in free_frame(). To resolve the page fault, you must do the following:

<ol>

 <li>Update the mapping from VPN to PFN in the current process’ page table.</li>

 <li>Invalidate the evicted process’ page table mapping.</li>

 <li>Unmap the corresponding frame table entry.</li>

</ol>

If the evicted page is dirty, you will need to swap it out and write its contents to disk. To do so, we provide a method called swap_write(), where you can pass in a pointer to the victim’s page table entry and a pointer to the frame in memory. Similarly, after you map a new frame to a faulting page, you should check if the page has a swap entry assigned, and call swap_read() if so.

Swap space effectively extends the memory of your system. If physical memory is full, the operating system kicks some frames to the hard disk to accommodate others. When the “swapped” frames are needed again, they are restored from the disk into physical memory.

<h1>8        Finishing a Process</h1>

If a process finishes, we don’t want it to hold onto any of the frames that it was using. We should release any frames so that other processes can use them. Also: If the process is no longer executing, can we release the page table?

As part of cleaning up a process, you will need to also free any swap entries that have been mapped to pages.

You can use swap_free() to accomplish this. Implement the function proc_cleanup() in paging.c.

<h1>9        Better Victim Selection</h1>

In section 7, we relied on the select_victim_frame() function to tell us which frame to choose as our

“victim”.

We have provided you with a default, inefficient page replacement algorithm that randomly selects a page to be evicted. The simulator can run this replacement strategy out-of-the-box so that you can test the other parts of your code without having to write a page replacement algorithm. Run the simulator with -rrandom to use the random algorithm.

Of course, we can do better than random replacement. Implement the <strong>least recently used replacement algorithm</strong>. Your LRU algorithm should choose the least recently accessed frame table entry based on a ”timestamp”, which represents the time that the entry was last accessed. Each frame table entry contains a timestamp field, which you will update whenever you make an access to the corresponding frame in memory. To get the current time of the simulator, you may use the provided get_current_timestamp() function.

<strong>Remember again that if the protected bit is set, it should never be chosen as a victim frame.</strong>

Once you have implemented the LRU algorithm, you will be able to run the simulator with the -rlru argument to use the algorithm as your page replacement strategy. Once you write your stats function in section 10, compare the performance of the two algorithms. What do you observe?

<h1>10        Computing AAT</h1>

In the final section of this project, you will be computing some statistics.

<ul>

 <li>writes – The total number of accesses that were writes</li>

 <li>reads – The total number of accesses that were reads</li>

 <li>accesses – The total number of accesses to the memory system</li>

 <li>page faults – Accesses that resulted in a page fault</li>

 <li>writebacks – How many times you wrote to disk</li>

 <li>aat – The average access time of the memory system</li>

</ul>

We will give you some numbers that are necessary to calculate the AAT:

<ul>

 <li>MEMORY READ TIME – The time taken to access memory <strong>SET BY SIMULATOR</strong></li>

 <li>DISK PAGE READ TIME – The time taken to read a page from the disk <strong>SET BY SIMULATOR</strong></li>

 <li>DISK PAGE WRITE TIME – The time taken to write to disk <strong>SET BY SIMULATOR</strong></li>

</ul>

You will need to implement the compute_stats() function in stats.c

<h1>11        How to Run / Debug Your Code</h1>

<h2>11.1       Environment</h2>

Your code will need to compile under Ubuntu 18.04 LTS. You can develop on whatever environment you prefer, so long as your code also works in Ubuntu 18.04 (which we will use to grade your projects). <strong>Noncompiling solutions will receive a 0! Make sure your code compiles with no warnings. </strong>We highly recommend using the Vagrant setup available on Canvas. Vagrant allows you to write code on your host machine and sync it with your VM through a shared folder. If you are having trouble with this, ask a TA about it in office hours.

<h2>11.2       Compiling and Running</h2>

We have provided a Makefile that will run gcc for you. To compile your code with no optimizations (which you should do while developing, it will make debugging easier) and test with the “random” algorithm, run:

$ make

$./vm-sim -i traces/&lt;trace&gt;.trace -rrandom

Once your LRU algorithm has been implemented, you can run the program with the -rlru argument in order to test. For example, you should run:

$ make

$./vm-sim -i traces/&lt;trace&gt;.trace -rlru

We highly recommend starting with “simple.trace.” This will allow you to test the core functionality of your virtual memory simulator without worrying about context switches or write backs, as this trace contains neither.

<h2>11.3       Corruption Checker</h2>

One challenge of working with any memory-management system is that your system can easily corrupt its own data structures if it misbehaves! Such corruption issues can easily hide until many cycles later, when they manifest as seemingly unrelated crashes later.

To help with detecting these issues, we’ve included a “corruption check” mode that aggressively verifies your data structures after every cycle. To use the corruption checker, run the simulator with the -c argument:

$./vm-sim -c -i traces/&lt;trace&gt;.trace -r&lt;algorithm&gt;

<h2>11.4       Debugging Tips</h2>

If your program is crashing or misbehaving, you can use GDB to locate the bug. GDB is a command line interface that will allow you to set breakpoints, step through your code, see variable values, and identify segfaults. There are tons of online guides, <a href="http://condor.depaul.edu/glancast/373class/docs/gdb.html#Setting_Breakpoints">click here</a> (http://condor.depaul.edu/glancast/373class/docs/gdb.html) for one.

To compile with debugging information, you must build the program with make debug:

$ make clean

$ make debug

To start your program in gdb, run:

$ gdb ./vm-sim

Within gdb, you can run your program with the run command, see below for an example:

$ (gdb) r -i traces/&lt;trace&gt;.trace -r&lt;algorithm&gt;

You may find it useful to set a breakpoint inside the main loop of the simulator to debug specific simulator commands in your implementation. You can do this either by finding the line number inside pagesim.c and breaking there:

$ (gdb) break pagesim.c:53 ! set breakpoint at call to system_init

$ (gdb) r -i traces/&lt;trace&gt;.trace -r&lt;algorithm&gt;

! (wait for breakpoint)

$ (gdb) s                                 ! step into the function call

or by using the actual function name being called from the main loop:

$ (gdb) break sim_cmd                                                 ! set breakpoint at call to sim_cmd

$ (gdb) r -i traces/&lt;trace&gt;.trace -r&lt;algorithm&gt;

! (wait for breakpoint)

$ (gdb) s                                 ! step into the function call

Sometimes, you may want to examine a large area of memory. To do this in GDB, you can use the x command (short for examine). For example, to examine the first 24 bytes of the frame table, we could do the following:

$ (gdb) x/24xb frame_table

0x1004000aa: 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00

0x1004000b2: 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00

0x1004000ba: 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00

The format of this command is x/nfu [memory location], where <strong>n </strong>is the number of items to print, <strong>f </strong>is a formatting identifier (for example, <strong>x </strong>for hexadecimal), and <strong>u </strong>is the specifier for the units you would like to print. <strong>b </strong>specifies units of 1 byte, <strong>h </strong>specifies 2 bytes, <strong>w </strong>specifies 4 bytes, and <strong>g </strong>specifies 8 bytes. So, our above command showed us 24 bytes of memory starting at frame_table in hexadecimal form.

If you use the corruption checker, you can set a breakpoint on panic() and use a backtrace to discover the context in which the panic occurred:

$ (gdb) break panic

$ (gdb) r -i traces/&lt;trace&gt;.trace -r&lt;algorithm&gt;

! (wait for GDB to stop at the breakpoint)

$ (gdb) backtrace

$ (gdb) frame N                                            ! where N is the frame number you want to examine

<strong>Feel free to ask about gdb and how to use it in office hours and on Piazza. Do not ask a TA or post on Piazza about a segfault without first running your program through GDB.</strong>

<h2>11.5       Verifying Your Solution</h2>

On execution, the simulator will output data read/write values. To check against our solutions, run

$ ./vm-sim -i traces/&lt;trace&gt;.trace -rlru &gt; my_output.log

$ diff my_output.log outputs/&lt;trace&gt;.log

You <strong>MUST </strong>implement the LRU algorithm in order to test against all the TA-generated output files.

We have also provided a “astar-random.log” file so that you may test your partial solution without needing to implement the LRU algorithm. To check against this output, run:

$ ./vm-sim -i traces/astar-random.trace -rrandom &gt; my_output.log

$ diff my_output.log outputs/astar-random.log

<strong>NOTE: To get full credit you must completely match the TA generated outputs for each trace.</strong>