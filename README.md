# embedded-linux-fs
Embedded Linux file system study : )

# Theory
- A file system is a kernel-space module in Linux.
- The Linux system includes a Virtual File System (VFS) layer.
  * VFS acts as an abstraction layer between user-space programs and various file systems.
  * To integrate with the VFS, a file system module must implement the VFS interface, which defines operations such as read, write, open, and close.
  * Those read, write, open, and close operations interact with physical devices such as SD cards or flash chips.
  * A file system should provide a formatting tool, which is a user-space program used to format storage devices. For example: `sudo mkfs.myfs /dev/mmcblk0p3` myfs is the filesystem. (/usr/local/bin/mkfs.myfs is the formatting tool) 

# How does Linux determine which file system (kernel module) to use when running `ls` in a folder?


# VFS
- superblock
- inode

# mkfs Command formatting tool
```
To make your utility work with mkfs -t myfs, you need to:
Name your binary mkfs.myfs.
Place it in a directory included in the PATH (e.g., /usr/local/bin).
The mkfs command looks for binaries named mkfs.<fsname> to perform formatting.
```

# List all the file systems. They are the installed kernal space modules. `nodev` means not physical device
```
cat /proc/filesystems
nodev   ramfs
nodev   rpc_pipefs
nodev   devpts
        ext3
        ext2
```

# Trying a read-only file system `squashfs`
