#OPERATING SYSTEM

&nbsp;

##Intro

&nbsp;

###Booting explained

&nbsp;

```
._____________.                      ._______________.
|             |                      |               |
| MAIN BOARD  |                      | HARD DRIVE    |
|             |______________________|               |_________
| - BIOS      |                      | - Partitions  |         \
|             |                      | (GRUB, files) |          \
|_____________|                      |_______________|           \
       |       \                                                  \
       |        \                                                  \
       |         \                                                  \
       |          \                   ._______________________.      |
       |           \                  |                       |      |
       |            \_________________|       CPU             |      |
       |                              |                       |      |
       |                              | - Registers           |      |
       |                              | (AX, CX)              |      |
       |                              | (stack pointer,       |      |
       |                              | instruction pointer)  |      |
       |                              |_______________________|      |
       |                                 /                          /
       |                                /                          /
.__________________________________________________.              /
|                                                  |             /
|                     RAM                          |            /
|                                                  |           /
| 1. Firmware from main board is copied to ram     |          /
| 2. Instruction pointer from CPU copied to ram    |         /
| 3. CPU reads instructions and executes firmware  |        /
| 4. Firmware asks the cpu to talk to the HD       |_______/
| 5. CPU loads the HD (bootloader part) into ram   |
| 6. Thanks the bootloader CPU can look into       |
|    other partitions from the hard drive          |
|    and will read /boot/grub/grub.cfg             |
| 7. Then it will load the kernel for the proper   |
|    operating system chosen from GRUB, kernel is  |
|    loaded from /boot/kernel(binary)              |
| 8. Now the operating system will boot            |
|                                                  |
|           RAM WILL BE LOADED WITH:               |
|           Firmware - GRUB - Kernel               |
|__________________________________________________|

```

NOTE: GRUB is here the main bootloader, but for making your own OS it is common to write your own bootloader.

&nbsp;

####Instruction pointer

&nbsp;

Also called the **program counter**, instruction address register or instruction counter,

https://en.wikipedia.org/wiki/Program_counter

&nbsp;

####Stack pointer

&nbsp;

The stack pointer stores the **address** of the most recent entry that was pushed onto the **stack**.

To push a value onto the stack, the stack pointer is incremented to point to the next physical memory address, and the new value is copied to that address in memory.

To pop a value from the stack, the value is copied from the address of the stack pointer, and the stack pointer is decremented, pointing it to the next available item in the stack.

The most typical use of a hardware stack is to store the return address of a subroutine call. When the subroutine is finished executing, the return address is popped off the top of the stack and placed in the Program Counter register, causing the processor to resume execution at the next instruction following the call to the subroutine.

&nbsp;

###Building OS

&nbsp;

When programming in C++, C++ expects the **stack pointer** to be set but the **bootloader** will not set the register as a stack pointer. To make this happen we need to make the linking. This
procedure exists out of 2 parts: **assembly code** and the **kernel**.

Software needed:

- GNU G++ (compiler)
- GNU Binutils (assembler and linker)
- libc6-dev-i386

```

.__________________.                      .________________.
|                  |                      |                |
| ASSEMBLY LOADER  |                      | KERNEL(cpp)    |
|                  | -------------------> |                |
| - Sets stack ptr |                      |                |
| - then jumps to  |                      |                |
|   the kernel     |                      |________________|
|__________________|                               |
         |                                         |                                                     
         |                                         |
         |                                         |
  compiled with GNU 'as'                    compiled with g++
         |                                         |                            
         |                                         |
         |                                         |
      loader.o                                  kernel.o
          \                                       /
           \                                     /
            \-> link them together using 'ld' <-/
                            |
                            |
                            |
                        kernel.bin

```

&nbsp;

####Show a UI

&nbsp;

Normally to print to the screen you should just use the c++ method **printf()**, but actually printf relies on **glibc** and there is no glibc when there is not yet an OS. So we need to write our "own" printf().

There is a specific RAM location **0xb8000** that will output everything that is in this location to the screen, thanks to the **graphics card**. The **color bytes** are there for foreground and background color. You just need to make sure you don't overwrite these bytes.

```

RAM:

0xb8000
._______._______._______._______._______._______.
|       |       |       |       |       |       |
| color |   a   | color |   b   |       |       |
| bytes |       | bytes |       |       |       |
|_______|_______|_______|_______|_______|_______|       
                   |
                   |
                   |
                   |
            .______|______.
            |             |
            |   SCREEN    |
            |             |
            | ab          |
            |             |
            |_____________|

```

&nbsp;

##Communicate with hardware

&nbsp;

To communicate with hardware everything needs to be **byte-perfect**.

To make this communication happen we need to write an **interrupt descriptor table** to talk to the **programmable interrupt controller**. An IDT contains the information for example for a keyboard interrupt so it will switch the memory segment to kernel space and switch the access rights. But we haven't defined what a **segment** is, this is done with a **global descriptor table**. A GDT will hold information about the segments, it will hold the starting point of a segment, the length of the segment and **flags** which holds data about the information of the segments (code, data, allowed to jump, executables, ..).

```

RAM:

           Kernel space                                      User space
 /-------------------------------\               /-------------------------------\
 | code segment  | data segment  |               | code segment  | data segment  |
_._______._______._______._______._______._______._______._______._______._______._
 |       |       |       |       |       |       |       |       |       |       |
 |       |       |       |       |       |       | code  |  code |       |       |
 |       |       |       |       |       |       |       |       |       |       |
_|_______|_______|_______|_______|_______|_______|_______|_______|_______|_______|_
   ^  |        \
   |  |         \
   |  |          \_______________
   |  |                          \
   |  |                       .___\___.
   |  |                       |       |
   |  |                       |  GDT  |
   |  |                       |_______|
   |  |                            \
   |  |                             \_________
   |  |                                       \                                  
   |  |                                        \
.__________________.                   .________________.      ._______.      .___________.
|                  |                   |                |      |       |      |           |
|      IDT         |                   |      CPU       | <----|  PIC  | <----| Keyboard  |
|                  | <---------------  |                |      |_______|      |___________|
| -Jump to k-space |                   |                |                
|                  |                   |________________|
|__________________|                               

```

&nbsp;

###GDT

&nbsp;

A GDT entry is **8 bytes** long, the laast 16 bits are for the length limit for example 1024 bytes. The first, fourth, fifth and sixth are for the pointer. The third byte is for access rights and the second is divided into two halfbytes, one for the length limit and one for the flags.

So this is pure chaos and you will need to fill the GDT manually.

```

     limit and flags
            |
._______._______._______._______._______._______._______._______.
|       |       |       |       |       |       |       |       |
|  PTR  | L | F | accss |  PTR  |  PTR  |  PTR  | lnght | lngth |
|       |   |   | rghts |       |       |       | limit | limit |
|_______|_______|_______|_______|_______|_______|_______|_______|

```

###PIC

&nbsp;

The programmable interrupt controller is a device that is used to combine several sources of interrupt onto one or more **CPU lines**. We have to send data to the PIC to obtain interrupts from the keyboard. Technically the CPU has a **multiplexer** and a **demultiplexer** which are connected to different hardware. You can put a number into a multiplexer and it will then talk to the hardware with that **port**, for example PIC has number 32 (0x20).

```

.__________________.                      .________________.        .___________.
|                  |                      |                |        |           |
|      CPU         |                      |      PIC       | <------| Keyboard  |
|                  | -----MPLEX---------> |                |        |___________|
|                  |        |             |                |                
|                  | -----DMPLEX--------> |________________|
|__________________|        |       |
                            |       |
                            |       |
                             \       \                
                              \       \                  
                             other hardware(hardware clock, mouse, ..)                

```


&nbsp;

###IDT

&nbsp;

TODO

```



```

&nbsp;

##Networking

&nbsp;

TODO

```


```