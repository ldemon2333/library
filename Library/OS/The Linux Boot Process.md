Truth is, this process is anything but simple. It carries behaviors, workarounds and quirks accumulated in almost 40 years of evolution. We call this process **boot**, short for "bootstraping" as in "pulling oneself up by one's bootstrap". The sense is, unlike the long chain of programs that are called by by other programs, the computer needs somehow call itself when turned on.

# Operational Modes
x86 computers have different operational modes, however we're interested in the main two: **real** and **protected**.

**Real mode** is the most basic operational mode and the mode the computer is set when it first starts. In this mode, programs have access only to the first megabyte of memory, but access to memory and I/O devices is completely unrestricted. All hardware assisted features such as memory protection, multitasking and protection rings are not available in real mode. This mode is where the first stages of the boot process happen, preparing the system to switch to **protected mode**.

**Real mode** is indeed the initial operating mode that a computer enters when it powers on or resets. It is the simplest mode of operation, typically associated with **x86 architecture**, and it has several key characteristics:

### Key Features of Real Mode:
1. **Memory Access Limitation**: 
   - Real mode allows the CPU to address a maximum of **1 MB** of memory (from **0x00000** to **0xFFFFF**). This is due to the 20-bit address bus, which provides 2^20 addressable memory locations (1,048,576 bytes, or 1 MB).
   
2. **Unrestricted Access to Hardware**:
   - In real mode, programs have **direct access** to hardware resources, including **memory** and **I/O devices**. There are no memory protection mechanisms, meaning one program can overwrite the memory of another, which can lead to system crashes or instability.
   
3. **Lack of Advanced Features**:
   - **Memory Protection**: Real mode does not provide memory protection, so there’s no isolation between programs, and programs can crash or corrupt memory arbitrarily.
   - **Multitasking**: Real mode doesn’t support hardware-assisted multitasking, meaning only one process can execute at a time. Time-sharing or multitasking requires protected or long mode (on x86 architectures).
   - **Protection Rings**: Real mode does not provide support for protection rings, so there is no concept of privileged or unprivileged operations (all code executes in ring 0).
   
4. **Boot Process**:
   - Real mode is where the first stage of the boot process occurs. When a computer is powered on, the **BIOS** (Basic Input/Output System) loads and starts executing in real mode. It initializes basic hardware components, such as the CPU, memory, and disk drives.
   - After this initial setup, the system may switch to **protected mode** (on more advanced systems) to enable more features like memory protection, hardware-assisted multitasking, and access to more than 1 MB of memory.

### Transition to Protected Mode:
- Once the basic hardware setup is complete in real mode, the system can transition to **protected mode** (in modern operating systems) or **long mode** (in 64-bit systems) to unlock the full capabilities of the CPU. In **protected mode**, memory can be addressed in larger quantities, and features like multitasking and memory protection become available.

### Why Real Mode Exists:
- Real mode was essential in the early days of computing, where processors like the **Intel 8086** and **8088** had limited addressing capabilities, and simpler systems didn’t need advanced features like memory protection.
  
In summary, **real mode** is a very basic and unprotected mode, where programs can interact directly with hardware but lack advanced features like memory protection or multitasking. It is the mode a system starts in before switching to more advanced operating modes such as **protected mode** or **long mode**.

**Protected mode** is the mode used by all modern operational systems. It supports the following features:
- **multitasking:** in protected mode, multiple programs can run at the same time, sharing the CPU time using preemptive multitasking.
- **paging:** to increase security and stability, each individual program access the physical memory using **virtual memory page tables**. Basically, each program sees a different memory table, these memory table **pages** (fixed-sized segments of 4096 bytes) are mapped to physical memory. Each mapped page has security flags that indicate whether the page can be shared with other process, is readable and/or writable, and holds data or executable code. These flags are used to prevent pages belonging to a program to be used or modified by another program.
- **protection rings:** to improve security, the program executable code is further segmented in different privilege levels called rings. We'll see more about it in a moment.

Protected memory allows up to 4 GB of memory to be addressed by a single program. An extension to protected mode, known as long mode, expands this limit to 256 TB.

# Protection Rings
Protected mode implements 4 distinct privilege levels ("rings"), numbered from 0 to 3, 0 being the most privileged and 3 the least. In practice though, Linux only uses rings 0 and 3, there is a non-negligible cost of switching between the rings:

- **Ring 0** is where the kernel, the device drivers and the system calls run. It has unrestricted access to memory and I/O devices. This mode of operation is known as kernel mode.
- **Ring 3** is where user programs and system libraries run. It can execute code, but it can't manipulate its own memory, access I/O devices or change the current protection ring (otherwise, it would be able to switch to ring 0). This mode of operation is known as user mode.

When a program running in user mode needs to access a restricted resource (like writing to disk or allocating more memory), it uses a system call. A system call is initiated generating a **software interrupt**; switching the content to kernel mode and checking the caller permissions for the requested operation. If the program has the required permissions, the system call code runs and the context is switched back to user mode.



![[Pasted image 20241114003406.png]]

# Basic Input Output System (BIOS)
The BIOS is a firmware used to perform the hardware initialization when the computer starts and provide low level runtime services for the operational system and user program. A set of instructions that is written on very low level language. BIOS is stored in a non-volatile memory called **Erasable Programmable Read-only Memory (EPROM)** chip and the configuration settings of BIOS is stored in the **Complementary Metal-Oxide Semiconductor (CMOS)** flash memory.

When the system boots, the firmware code is mapped to the memory and the CPU starts in real mode. Since CPU have no way to know beforehand where to look for the actual firmware code, so it always execute the code contained in the fixed physical memory address **000FFFF0**. Normally, this address contains a JMP instruction, pointing to the actual BIOS code.

![[Pasted image 20241114003911.png]]
==When the computer is powered on, the CPU approaches the BIOS to check if all the basic hardware required to start the computer is ready. This checking is known as the== ==**Power of Self Test (POST)**==. POST process only happens when performing a so-called “_cold boot_” (turning on a powered off system); a “_warm boot_” (i.e. restarting the computer) leaves a special flag in the BIOS non-volatile memory that bypasses this task of checking.

The BIOS code then performs two tasks:

- **POST (power on self-test):** identifies and initialize the connected hardware devices, such as CPU, memory, video cards and disk controllers. Some devices have their own firmware; the POST process identifies these devices and execute their code as well. POST process only happens when performing a so-called "cold boot" (turning on a powered off system); a "warm boot" (i.e. restarting the computer pressing `<ctrl>+<alt>+<del>`) leaves a special flag in the BIOS non-volatile memory that bypasses this task.
- **Boot:** once POST is complete, the BIOS calls INT 19h to start the boot process. The POST code is unloaded from memory and the BIOS looks for the boot devices as configured in its non-volatile memory. The process of booting from disk works loading its first sector to memory (known as **master boot record, MBR**) and trying to execute it. If it fails, it tries the next device until it runs out; if no device able to boot is found, it gives the user an error and stops.

加电自检针对：
- Hardware elements like processor, storage devices and memory.
- Basic System Devices like keyboard, and other peripheral devices.
- CPU Registers
- DMA (Direct Memory Access)
- Timer
- Interrupt controller

When POST is successfully finalized, bootstrapping is enabled. Bootstrapping starts the initialization of the OS. The process of booting from disk works by loading its first sector to memory known as **Master Boot Record (MBR)** and trying to execute it. If it fails, it tries the next device until it runs out; if no device able to boot is found, it gives the user an error and stops. The first boot sector it finds that contains a valid boot record is loaded into RAM and control is then transferred to the code that was loaded from the boot sector.

# Master Boot Record (MBR)
_Master Boot Record (MBR)_ or the primary bootloader is the first sector of the **Hard Disk Drive (HDD)**. Its physical location is in the cylinder 0, head 0, sector 1. The MBR is only 512 bytes in size and contains machine code instructions for booting the machine, called a boot loader, along with the partition table. The first 446 bytes are the primary boot loader, which contains both executable code and error message text. The next 64 bytes are the partition table, which contains a record for each of 4 partitions (16 Bytes each). The MBR ends with 2 bytes that are defined as the magic number (0xAA55). The magic number serves as a validation check of the MBR.

![[Pasted image 20241114005031.png]]
==The job of _MBR_ is to find and load the secondary boot loader==. It does this by looking through the partition table for an *active partition*. When it finds an active partition, it scans the remaining partitions in the table to ensure that they’re all inactive. ==When this is verified, the active partition’s boot record is read from the device into RAM and executed.

# Boot Loader (GRUB 2)
The execution context provided by the BIOS is too restrictive to load the operational system directly. To help the Linux kernel to load, a special program called boot loader is used. The most used boot loader in Linux system is GRUB 2.

GRUB 2 is too big to fit in 446 bytes, all but the most simple boot loaders are. To locate and load the kernel, it needs to support dozens of file systems and features such as encryption, software RAID and LVM. The space just not enough.

To overcome this limitation, GRUB 2 code is divided in stages, the early stage enables the load of the following stage and so on. This is what it looks like:

![[Pasted image 20241114110027.png]]

==The secondary bootloader is called *the kernel loader*. The task of this is to load the Linux kernel and optional initial RAM disk.== The first- and second-stage boot loaders combined are called **GRand Unified Bootloader (GRUB)**. GRUB includes knowledge of Linux file systems. 而不是使用磁盘上的原始扇区, as LILO does, GRUB can load a Linux kernel from an ext2 or ext3 file system. It does this by making the two-stage boot loader into a three-stage boot loader. Stage 1 (MBR) boots a stage 1.5 boot loader that understands the particular file system containing the Linux kernel image.

GRUB cannot fit in the 446 bytes of the MBR. To locate and load the kernel, GRUB needs to support dozens of file systems and features such as encryption, software RAID and LVM.

GRUB loads itself into memory in the following stages:

**The Stage 1**: this is the bare minimal code necessary to boot the next stage. It's not aware of partitions, files or file systems; all it has is a [LBA48](https://en.wikipedia.org/wiki/Logical_block_addressing#LBA48) pointer to the following stage. The BIOS translates the LBA48 pointer to the physical address of the sector to load from; data is loaded direct to memory and executed.

**The Stage 1.5**: it's used when Stage 1 can't access `core.img` directly from **Stage 2** (for example, the `/boot` partition is stored in a encrypted device). It uses the empty space between the MBR and the first disk partition (sector 63), therefore it's 32,256 bytes long. Unlike **Stage 1**, it's aware of file systems and it loads Stage 2 using its full filename. Since there is not enough room to keep all possible file system drivers, it's dynamically built to hold all it needs to access `/boot`.

**The Stage 2** or secondary boot loader is read into memory. It displays a list of available kernels (defined in `/etc/grub.conf`, with soft links from `/etc/grub/menu.lst` and `/etc/grub.conf`). This interface allows the user to select which kernel or operating system to boot, pass arguments to the kernel, or look at system parameters. Stage 2 loads the configuration file and any driver from the file system. This is the stage that shows an text-based selection menu and allow the user to customize the boot options.

GRUB 2 can be used to load non-Linux operational systems, such as Windows. It presents a menu that allow the user to chose the operational system, in Linux case, multiple kernel versions can coexist in the same installation and can be selected in the same way. Each Linux system/kernel option will define the following arguments:
- Kernel compressed image filename and path
- Root file system (as understood by the kernel)
- Optional initial RAM disk compressed image
- Optional kernel arguments

The secondary boot loader reads the operating system or kernel as well as the contents of /boot/sysroot/ into memory. Once GRUB determines which operating system or kernel to start, it loads it into memory and transfers control of the machine to that operating system.

# Kernel
Unfortunately, booting the kernel is not as simple as it seems. There are two main reasons:

- The kernel starts in real mode, not protected mode. Since the real mode can't access more than 1 megabyte of memory, it has not enough memory to full load the kernel.
- The kernel image data is compressed, only a small piece of the header is actual executable code. The header is kept under real mode memory limit, the remaining kernel is stored above and it's not accessible until the kernel switches to protected mode.

The kernel boot process follows these steps:
1. 检查 boot manager 是否按照内核引导协议所要求的布局加载了代码
2. Detect memory layout, validates the CPU architecture and set up the keyboard and the terminal video mode.
3. Prepare the system to switch to protected mode. It needs to perform two tasks: initialize a new **Interrupt Handler Table** and a **Global Desciptor Table**, both required by protected mode.
4. Switch to protected mode and decompress the kernel. The kernel is decompressed in place: the memory pages holding the compressed image are replaced by the decompressed data. This is where you see the familiar messages "_Decompressing Linux… done._" and "_Booting the kernel._"
5. 1. The actual kernel starts executing. Paging is enabled, interrupt handler table is initialized and [start_kernel()](https://elixir.bootlin.com/linux/v4.18.5/source/init/main.c#L531) is called. `start_kernel()` and the subsequent called functions have the process id 0, they are the ancestor of all kernel mode processes as well process id 1.
6. `start_kernel()` initializes dozens of kernel subsystems. Among them, the task scheduler. It ends calling [rest_init()](https://elixir.bootlin.com/linux/v4.18.5/source/init/main.c#L397).
7. `rest_init()` in its turn, spawns the very first user space process: `kernel_init()`. Its process id is 1 and it's special: it will become the direct or indirect ancestor of all user space processes. It also spawns [kthreadd](https://elixir.bootlin.com/linux/v4.18.5/source/kernel/kthread.c#L557) process (normally, process id 2), that's the parent of all kernel threads. Finally, it runs `cpu_idle()`, a process that takes over the CPU whenever there is no other process using it.
8. [kernel_init()](https://elixir.bootlin.com/linux/v4.18.5/source/init/main.c#L1057) will start any additional CPU core. If there is a initial RAM disk is defined, it will decompress and mount it. Then, it load the device drivers based on the BIOS [ACPI tables](https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface), mount the root file system in read-only mode and finally call the [init](https://en.wikipedia.org/wiki/Init) process (normally, in `/sbin/init`). As long as the computer is running, the init process remains the process id 1.

All of the kernels are in a self-extracting, compressed format to save space. The kernels are located in the /boot directory, along with an initial RAM disk image, and device maps of the hard drives.

After the selected kernel is loaded into memory and begins executing, it must first extract itself from the compressed version of the file before it can perform any useful work. Once the kernel has extracted itself, it loads systemd, which is the replacement for the old SysV init program, and turns control over to it.

This is the end of the boot process. At this point, the Linux kernel and systemd are running but unable to perform any productive tasks for the end user because nothing else is running.

# Initial RAM disk
The initial RAM disk is an optional resource used by the kernel boot process. It's a compressed disk image including kernel modules and command-line tools required by the kernel to mount the root file system. For example, if the root file system is a remote NFS exported directory, the initial RAM disk needs the network interface device drivers modules as well a DHCP client to retrieve the system IP address and [portmap](https://linux.die.net/man/8/portmap) daemon. Tools for troubleshooting a system are included as well.

When the kernel initialize, it looks for the expected memory locations for `initrd_address`and `initrd_size` (set by the boot loader, following the [kernel boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt) guidelines). If found, the correspondent pages are mapped to the special device file `/dev/initrd` (read-only). What happens next depends on the image format:

- **initrd:** fixed-size block device image (optionally compressed) where the the files are stored inside a file system. The kernel copies it to `/dev/ram0` and mount it as root. If the the `/linuxrc` file exists and is executable, it's executed by the kernel. Finally, the kernel tries to mount the root device (specified by the kernel command-line argument **root**, passed by the boot loader). Once mounted, it calls `[pivot_root()](http://www.tutorialspoint.com/unix_system_calls/pivot_root.htm)`to switch the root file system. The **initrd** image will be mounted under `/initrd` if the directory exists in the new root, otherwise the contents of `/dev/initrd` and `/dev/ram0` are discarded and the memory is reclaimed. Finally, the kernel will run the [init](https://en.wikipedia.org/wiki/Init) process (usually `/sbin/init`, can be configured with the kernel command-line argument **init**)
- **initramdisk:** CPIO archive (optionally compressed). The kernel extracts it directly to the file system (no need to mount a block device), then run `/init`. Unlike **initrd**, the kernel won't run the [init](https://en.wikipedia.org/wiki/Init) process automatically nor mount and switch the root file system: **initramdisk** must do it itself. Also, unlike **initrd**, the image is not available after switching to the real root device.

# Init Process
The final step in the boot process is when `kernel_init()`finishes and give place to the init process. Linux has at least three mainstream implementations of init process:

- **SysV init:** the classic implementation of init process, currently considered obsolete.
- **Upstart:** written by [Canonical](https://en.wikipedia.org/wiki/Canonical_Ltd.), was the first mainstream replacement for SysV init. Besides handling the initialization, its event-based infrastructure continues managing the system after the initialization: for example, it will automatically mount an USB flash drives when it's connected.
- **systemd:** the current _de facto_ standard on modern Linux distributions. Unlike **SysV init** and **Upstart**, its configuration describes the state the system must be after the initialization instead simply providing the scripts to reach that state.

While this article won't go deep on init, a list of some of most important tasks performed by it follows:

- Check the root file system and remount it in read/write mode. Up to this point, root file system is mounted in read-only mode.
- Check and mount additional file systems.
- Initialize network cards. Some processes are triggered by the network card initialization, such **ntpdate** (used to synchronize local time with a network time server).
- Start daemons. Some daemons support the system processes, such as **crond** and **syslogd**. Some allow remote access, such as **sshd** and **telnetd**. And some represent actual network services such as **named** and **httpd**.
- Sets [**getty**](https://en.wikipedia.org/wiki/Getty_(Unix)) on physical or virtual terminals. This allows the user connected to a terminal to login and use the shell.


# Init
After the kernel is booted and initialized, the kernel starts the first user-space application. This is the first program invoked that is compiled with the standard C library. Prior to this point in the process, no standard C applications have been executed.

In a desktop Linux system, ==the first application started is commonly /sbin/init.== But it need not be. Rarely do embedded systems require the extensive initialization provided by _init_ (as configured through /etc/inittab). In many cases, you can invoke a simple shell script that starts the necessary embedded applications.

Some of most important tasks performed by init are as following:

- Check the root file system and remount it in read/write mode. Up to this point, root file system is mounted in read-only mode.
- Check and mount additional file systems.
- Initialize network cards. Some processes are triggered by the network card initialization, such _ntpdate_ (used to synchronize local time with a network time server).
- Start daemons. Some daemons support the system processes, such as _crond_ and _syslogd_. Some allow remote access, such as _sshd_ and _telnetd_. And some represent actual network services such as _named_ and _httpd_.
- Sets _getty_ on physical or virtual terminals. This allows the user connected to a terminal to login and use the shell.

![[Pasted image 20241114011345.png]]

After the _sysinit.target_ is fulfilled, systemd next starts the _basic.target_, starting all of the units required to fulfill it. The basic target provides some additional functionality by starting units that re required for the next target. These include setting up things like paths to various executable directories, communication sockets, and timers.

When one of these targets is reached, then startup has completed. If the _multi-user.target_ is the default, then a text mode login is displayd on the console. If _graphical.target_ is the default, then a graphical login is diplayed; the specific GUI login screen will depend on the default display manager that is used.


![[Pasted image 20241114011506.png]]