# 1. What is RDMA ?
RDMA is Remote Direct Memory Access. 
But not all of engineers can understand it well because RDMA itself is very difficult to understand. I wrote it down step by step to understand it. 
Before talking about RDMA, we should study how DMA works and its problems. 
See also the URL below:
(https://github.com/developer-onizuka/what_is_DMA/blob/main/README.md)

# 2. Problems of traditional DMA
DMA is a copy between host memory and device's memory. As you can see below, the copy from device's BAR space to a physical memory address in host machine. After DMA between host and device, Device's DMA Engine interrupts to CPU so that CPU can start copying this DMAed data (it's still in kernel space) to the user space by CPU load, which is not DMA. Next, the space in kernel is released for the next DMA operation, which we know it as flow control of DMA.

Please note the copy from the kernel space to the user space was done by some processes with CPU load. It means the copy was performed thru "Virtual Address". 
Of cource, copy needs "Physical Address". Who translates the address from Virtual address to Physical address ??? Yes, Kernel does.

For copying after DMA, some processes (TCP/IP and NIC driver) call kernel API for translation from Virtual address to Physical address because Only the kernel knows the mapping between Virtual Address and Physical Address. After this translation, physical copies will be perform by hardware.

Guess how many interrupts happens if you are using so high throughput NIC? Today, there are so many High performance NICs from Mellanox and Intel... 
The problem is these heavy interrupts itself which introduce additional CPU workloads. CPU is not dedicated to I/O such as NIC but for the purpuses including user program's processing. In addtion, its resources such as TLB are very very precious. 
   
```
          Physical Memory
          +----------+
          |          |
          |          |
          +----------+
          |XXXXXXXXXX|
   +----- |XXXXXXXXXX|
   |      +----------+ 0xf0000000 (NIC BAR#1)
   |      |          |                                          Kernel Space (Virtual Address)
 Copy     |          |                                          +----------+
 (DMA)    |          |                                          |          |
   |      +----------+                                          |          | 
   +----> |XXXXXXXXXX|                                          |XXXXXXXXXX|
          |XXXXXXXXXX| <================Mapping===============> |XXXXXXXXXX| -----+
          +----------+ Host Memory for DMA operation            +----------+      |
          |          | (Physical Address)                                        Copy
          |          |                                          +----------+     (CPU)
          +----------+                                          |          |      |
          |XXXXXXXXXX|                                          |XXXXXXXXXX|      |
          |XXXXXXXXXX| <================Mapping===============> |XXXXXXXXXX| <----+
          +----------+ User Space (Physical Address)            +----------+
          |          |                                          User Space (Virtual Address)
          |          |
          +----------+ 0x00000000
```

# 3. Zero Copy (Kernel Bypass)
How can we come over the problem of traditional DMA? One of ideas is bypassing kernel so that any context switch never happens. The user application is programmed through virtual address because users can not refer the physical address. The user program can let the RDMA device such as InfiniBand HCA or Mellanox NIC  know user space's physical address by using "verbs API" in the user program. But RDMA is one of DMA and the difference is what it can translate the address itself by its hardware logic.

Step 1. 
---
User program creates its own space as virtual address thru malloc().
```
          Physical Memory
          +----------+            
          |          |             
          |          | 
          +----------+ 
          |          |
          |          | 
          +----------+ 0xf0000000 (NIC BAR) 
          |          |                       
          |          |
          |          | 
          |          | 
          |          | 
          |          |
          |          |
          |          |                                   
          |          |                                          +----------+
          |          |                                          |          |
          |          |                                          |          |
          |          |                                          |          |
          |          |                                          +----------+ 0xc0004000
          |          |                                          User Space (Virtual Address)
          |          |
          +----------+ 0x00000000
```
Step 2. 
---
User program asks the space registerd thru verbs API so that the kernel could not swap it out to disk. We call it "PIN". User program makes some controll resources. One of resource is PTE which is for translation table between physical address and virtual address of user program space. We call this operation "Memory Registration". PTE gonna be created in the background of the operation of Memory Registration. But please understand the Memory Registration is very heavy operation, so user should understand it for tuning performances. These are almost everything which the user program should do for perspectives of kernel bypass before a packet arriving.
```
          Physical Memory
          +----------+            
          |          |             
          |          |                                          RDMA Controll resources
          +----------+                                          +----------+
          |          |                                          |          |
          |          |                                          |          |
          +----------+ 0xf0000000 (NIC BAR)                     | PTE#1    |
          |          |                                          +----------+                       
          |          |
          |          | 
          |          | 
          |          | 
          |          |
          |          |
          |          |                                          PINNED
          |          |                                          +----------+
          +----------+                                          |          |
          |          |                                          |          |
          |          | <================Mapping===============> |          |
          +----------+ User Space (Physical Address)            +----------+ 0xc0004000
          |          |                                          User Space (Virtual Address)
          |          |
          +----------+ 0x00000000
```

Step 3. 
---
Data comes into the NIC logic and put it on BAR space on NIC. But the DMA engine doesn't know where it should do DMA to!
```
          Physical Memory
          +----------+            +---------- Data From Outside
          |          |            |
          |          |            |                             RDMA Controll resources
          +----------+            |                             +----------+
          |          |            |                             |          |
   +----- |XXXXXXXXXX| <----------+                             |          |
   |      +----------+ 0xf0000000 (NIC BAR#1)                   | PTE#1    |
   |      |          |                                          +----------+                       
   V      |          |
  ???     |          | 
          |          | 
          |          | 
          |          |
          |          |
          |          |                                          PINNED
          |          |                                          +----------+
          +----------+                                          |          |
          |          |                                          |          |
          |          | <================Mapping===============> |          |
          +----------+ User Space (Physical Address)            +----------+ 0xc0004000
          |          |                                          User Space (Virtual Address)
          |          |
          +----------+ 0x00000000
```
Step 4. 
---
DMA Engine starts fetching the PTE#1 from certain space from host memory so that it can know where it does DMA. Then, DMA Engine can copy between NIC and user space without addtional copys. But how does the user space understand the completion of copy without kernel interventions??? This is a new problem while we  don't use kernel features which we call "interrupts".
```
          Physical Memory
          +----------+            +---------- Data From Outside
          |          |            |
          |          |            |                             RDMA Controll resources
          +----------+            |                             +----------+
          |          |            |                             |          |
   +----- |XXXXXXXXXX| <----------+                             |          |
   |      +----------+ 0xf0000000 (NIC BAR#1)                   | PTE#1    |
   |      |          |                                          +--+-------+                       
   V      |          |                                             |
  ??? <------------------------------------------------------------+ 
   |      |          | 
   |      |          | 
   |      |          |
   |      |          |
   |      |          |                                          PINNED
   |      |          |                                          +----------+
   |      +----------+                                          |          |
   |      |          |                                          |          |
   +----> |XXXXXXXXXX| <================Mapping===============> |XXXXXXXXXX|
          +----------+ User Space (Physical Address)            +----------+ 0xc0004000
          |          |                                          User Space (Virtual Address)
          |          |
          +----------+ 0x00000000
```
Step 5. 
---
DMA Engine creates the Completion Queue(CQE#1) instead of interruptting to the kernel. User program polls until a CQE is created. This is the end of RDMA step. After completion, the data of BAR space can be removed (by incrementing the pointer) and NIC's receiver is ready for receiving next packets. Each resource in host memory such as PTE or CQE will be destroyed and unpinned if subsequent process does not use these resources. The space already registered may be used again so that we can prevent from heavy process of memory registration.
```
          Physical Memory
          +----------+            
          |          |             
          |          |                                          RDMA Controll resources
          +----------+                                          +----------+
          |          | <--- next pointer to receive             |          |
          |XXXXXXXXXX|                                          | CQE#1    | <---Polling by
          +----------+ 0xf0000000 (NIC BAR#1)                   | PTE#1    |     user program
          |          |                                          +----------+                       
          |          |                                              
          |          |
          |          | 
          |          | 
          |          |
          |          |                                          
          |          |                                          Unpinned (Kernel can swap out)
          |          |                                          +----------+
          +----------+                                          |          |
          |          |                                          |          |
          |XXXXXXXXXX| <================Mapping===============> |XXXXXXXXXX|
          +----------+ User Space (Physical Address)            +----------+ 0xc0004000
          |          |                                          User Space (Virtual Address)
          |          |
          +----------+ 0x00000000
```
