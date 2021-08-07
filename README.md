# 1. What is RDMA ?
RDMA is Remote Direct Memory Access. 
But not all of engineers can understand it well because RDMA itself is very difficult to understand. I wrote it down step by step to understand it. 
Before talking about RDMA, we should study how DMA works between devices and the problems. 
See also the URL below:
(https://github.com/developer-onizuka/what_is_DMA/blob/main/README.md)

# 2. Problems of traditional DMA:
DMA is a copy between host memory and device's memory. As you can see below, the copy from device's BAR space to a physical memory address in host machine.  
After DMA between host and device, Device's DMA Engine interrupts to CPU so that CPU can start copying this DMAed data (it's still in kernel space) to the user space by CPU load, which is not DMA. Next, the space in kernel is released for the next DMA operation, which we know it as flow control of DMA.

Please note the copy from the kernel space to the user space was done by some processes with CPU load. It means the copy was performed thru "Virtual Address". 
Of cource, copy needs "Physical Address". Who translates the address from Virtual address to Physical address ??? 

Before copying, some processes call kernel API for translation from Virtual address to Physical address because Only the kernel knows the mapping between Virtual Address and Physical Address. After this translation, physical copies will be perform by hardware.

   
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
          |XXXXXXXXXX|                                          |XXXXXXXXXX| <----+
          |XXXXXXXXXX| <================Mapping===============> |XXXXXXXXXX|
          +----------+ User Space (Physical Address)            +----------+
          |          |                                          User Space (Virtual Address)
          |          |
          +----------+ 0x00000000
```
