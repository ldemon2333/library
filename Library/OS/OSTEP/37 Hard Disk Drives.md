# 37.1 The Interface
让我们首先了解现代磁盘驱动器的接口。所有现代驱动器的基本接口都很简单。驱动器由大量扇区（512 字节块）组成，每个扇区都可以读取或写入。在具有 n 个扇区的磁盘上，扇区编号从 0 到 n - 1。因此，我们可以将磁盘视为扇区数组；0 到 n - 1 是驱动器的地址空间。

多扇区操作是可能的；事实上，许多文件系统每次都会读取或写入 4KB（或更多）。但是，在更新磁盘时，驱动器制造商唯一能保证的是，单个 512 字节写入是原子的（即，要么全部完成，要么根本无法完成）；因此，如果发生不合时宜的断电，则较大的写入只有一部分可能完成（有时称为残缺写入）。

大多数磁盘驱动器客户端都会做出一些假设，但这些假设并未在接口中直接指定；Schlosser 和 Ganger 将此称为磁盘驱动器的“不成文合同”。具体而言，

When updating the disk, the only guarantee drive manufactures make is that a single 512-byte write is **atomic**. There are some assumptions most clients of disk drives make, but that are not specified directly in the interface. 
- One can usually assume that accessing two blocks near one-another within the drive's address space will be faster than accessing blocks that are far apart.
- One can also usually assume that accessing blocks in a contiguous chunk is the fastest access mode, and usually much faster than any more random access pattern.

# 37.2 Basic Geometry
我们先从盘片开始，盘片是一个圆形的硬表面，通过对其产生磁性变化，数据可以持久地存储在其上。磁盘可能有一个或多个盘片；每个盘片有 2 个侧面，每个侧面称为表面。这些盘片通常由某种硬质材料（例如铝）制成，然后涂上一层薄薄的磁性层，使驱动器即使在断电时也能持久存储数据。

所有盘片都绑在主轴上，主轴连接到一个电机，电机以恒定（固定）的速率旋转盘片（当驱动器通电时）。旋转速率通常以每分钟转数 (RPM) 来衡量，典型的现代值在 7,200 RPM 到 15,000 RPM 范围内。请注意，我们通常会对单次旋转的时间感兴趣，例如，以 10,000 RPM 旋转的驱动器意味着单次旋转大约需要 6 毫秒 (6 ms)。

数据以同心圆的形式编码在每个表面上的扇区中；我们将这样的一个同心圆称为一个轨道。单个表面包含成千上万个紧密堆积在一起的轨道，数百个轨道的宽度与人类头发的宽度相当。

要从表面读取和写入，我们需要一种机制，使我们能够感知（即读取）磁盘上的磁性图案或引起磁性图案的变化（即写入）。此读取和写入过程由磁盘头完成；驱动器的每个表面都有一个这样的磁盘头。磁盘头连接到单个磁盘臂，该磁盘臂在表面上移动以将磁盘头定位在所需的轨道上。

# 37.4 I/O Time: Doing The Math
We can now represent I/O time as the sum of three major components:
![[Pasted image 20241201165323.png]]


