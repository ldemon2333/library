https://learn.microsoft.com/en-us/windows/wsl/disk-space

WSL2 使用虚拟化平台，创建一个VHD格式来存储Linux文件。这些VHD使用ext4 文件系统，在 Windows 硬盘驱动器上表示为 ext4.vhdx 文件。

WSL2自动调整这些 VHD 文件的大小以满足存储需求。默认情况下，WSL 2使用的每个 VHD 文件最初都被分配了1TB 的最大磁盘空间量(在 WSL 发布0.58.0之前，这个默认值被设置为512 GB max 和256 GB max)。

To check available disk space, open a PowerShell command line and enter this command (replacing `<distribution-name>` with the actual distribution name):

```
wsl.exe --system -d <distribution-name> df -h /mnt/wslg/distro
```


The virtual disk that WSL2 uses (`ext4.vhdx`) is what is known as "sparse". In other words, it appears to the underlying OS as its maximum available size, but it only takes up as much space on the host (Windows) disk as it currently needs.

You can find this file in Windows in `%userprofile%\AppData\Local\Packages\Canonical..\LocalState\ext4.vhdx`.

请注意，空间，一旦消耗，不会自动释放。例如，在 WSL/Ubuntu 中创建一个1GB 的文件将使虚拟磁盘映像的大小增加1GB。然而，删除该文件不会恢复 Windows 中的空间(但在 Ubuntu 中会恢复)。

There are ways to [manually shrink it](https://superuser.com/a/1612289/1210833?_gl=1*ywbo2z*_ga*NTM0NzEzMTA5LjE3Mjk4NDU1NTE.*_ga_S812YQPLT2*MTcyOTg0NTU1MC4xLjAuMTcyOTg0NTU1MC4wLjAuMA..) if needed to reclaim space on the Windows drive.


https://superuser.com/questions/1606213/how-do-i-get-back-unused-disk-space-from-ubuntu-on-wsl2/1612289#1612289


# Q：
I'm using Ubuntu 20.04 on WSL2 on Windows 10, and I noticed that after removing files on Ubuntu I was not getting the space back that was taken up by the removed files. 

So how can I get back those unused bits? Is there some terminal command which I can use or can I do something via the windows GUI or something?

#### "Original" Answer (Preferred)

There's a WSL Github [issue](https://github.com/microsoft/WSL/issues/4699) open on this topic. WSL will automatically grow the virtual disk (ext4.vhdx), but shrinking it to reclaim unused space is something that must currently be done manually.

The first thing you'll need to do is know the location of your ext4.vhdx. For a default Ubuntu installation, it should be in something like `%USERPROFILE%\AppData\Local\Packages\CanonicalGroupLimited.UbuntuonWindows_79rhkp1fndgsc\LocalState\`

Then there are several techniques that you can use to remove the unused space. I recommend you start with a `wsl --shutdown` and copy the vhdx as a backup to start. If you are running Docker Desktop, also shut it down, otherwise it may inadvertently attempt to restart WSL after your `--shutdown`.

- If you are on Windows Professional or higher, you can install Hyper-V and use the Optimize-VHD commandlet as described in the [original issue](https://github.com/microsoft/WSL/issues/4699). .
    
- On Windows Home (and higher) you can use `diskpart` as described [in this comment](https://github.com/microsoft/WSL/issues/4699#issuecomment-627133168).
    
- Exporting the WSL distro and re-importing it into a new WSL instance (as in [this comment](https://github.com/microsoft/WSL/issues/4699#issuecomment-660104214)) will also reclaim the space. Note that you will need to reset the default username after an import. See [here](https://github.com/microsoft/WSL/issues/4276#issuecomment-553367389) (and [here](https://superuser.com/a/1627461/) for alternative options).
    

I have tested and confirmed both the second and third techniques personally.


第二种方法
![[Pasted image 20241025165227.png]]
