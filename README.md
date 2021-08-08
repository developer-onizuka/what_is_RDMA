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

Before copying, some processes call kernel API for translation from Virtual address to Physical address because Only the kernel knows the mapping between Virtual Address and Physical Address. After this translation, physical copies will be perform by hardware.

Guess how many interrupts happens if you are using so high throughput NIC ? Today, there are so many High performance NICs from Mellanox and Intel... 
The problem is heavy interrupts which introduce additional CPU workloads. CPU is not only dedicated to I/O such as NIC and its resources such as TLB are very very precious. 
   
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

# 3. Zero Copy
How can we come over the problem of traditional DMA ? One of ideas is bypassing kernel so that any context switch never happens. The user application is programmed through virtual address because users can not refer the physical address.

InfiniBand or some other RDMA NIC 
RDMA is one of DMA

Step 1. User program creates its own space as virtual address thru malloc().
---
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
          +----------+                                          |          |
          |          |                                          |          |
          |          |                                          |          |
          +----------+ User Space (Physical Address)            +----------+ 0xc0004000
          |          |                                          User Space (Virtual Address)
          |          |
          +----------+ 0x00000000
```
Step 2. User program asks the space registerd thru verbs API so that the kernel could not swap it out to disk. We call it "Pin".
---
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
Step 3. User program makes some controll resources. One of resource is PTE which is for translation table between physical address and virtual address of user program space.
---
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

Step 1. Data comes into the NIC logic and put it on BAR space on NIC.
```
          Physical Memory
          +----------+            +---------- Data From Outside
          |          |            |
          |          |            |                             Kernel Space (RDMA Controll resources)
          +----------+            |                             +----------+
          |          |            |                             |          |
          |XXXXXXXXXX| <----------+                             |          |
          +----------+ 0xf0000000 (NIC BAR#1)                   |          |
          |          |                                          +----------+                       
          |          |
          |          | 
          |          | 
          |          | 
          |          |
          |          |
          |          |                                   
          |          |                                          +----------+
          +----------+                                          |          |
          |          |                                          |          |
          |          |                                          |          |
          +----------+ User Space (Physical Address)            +----------+
          |          |                                          User Space (Virtual Address)
          |          |
          +----------+ 0x00000000
```

Step 2.
```
          Physical Memory
          +----------+
          |          |
          |          |                                          Kernel Space (IB resource)
          +----------+                                          +----------+
          |XXXXXXXXXX|                                          |          |
   +----- |XXXXXXXXXX|                                          |          |
   |      +----------+ 0xf0000000 (NIC BAR#1)                   |          |
   |      |          |                                          +----------+                       
 Copy     |          |
 (DMA)    |          | 
   |      |          | 
   |      |          | 
   |      |          |
   |      |          |
   |      |          |                                   
   |      |          |                                          +----------+
   |      +----------+                                          |          |
   |      |XXXXXXXXXX|                                          |XXXXXXXXXX|
   +----> |XXXXXXXXXX| <================Mapping===============> |XXXXXXXXXX|
          +----------+ User Space (Physical Address)            +----------+
          |          |                                          User Space (Virtual Address)
          |          |
          +----------+ 0x00000000

```
