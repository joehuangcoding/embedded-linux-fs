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


# Commands for kernel modules
```
make clean
make
sudo insmod pseudo_fs.ko
dmesg | tail -n 50

lsmod | grep pseudo_fs

//mount the file system
sudo mount -t pseudo_fs none /mnt
sudo umount /mnt
//Checking if in used
sudo lsof /mnt
ps aux | grep /mnt
sudo umount -f /mnt
sudo kill -9 <PID>


//Uninstall
sudo rmmod pseudo_fs

```

# Code for testing - need some fixes
```
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/init.h>
#include <linux/pagemap.h>
#include <linux/kernel.h>
#include <linux/string.h>
#include <linux/timekeeping.h> // For current_time()

#define PSEUDOFS_MAGIC 0x12345678
#define PSEUDO_FILE_CONTENT "Hello from pseudo file system!\n"
#define PSEUDO_FILE_NAME "hello.txt"

// Pseudo file operations
static ssize_t pseudo_read(struct file *file, char __user *buf, size_t len, loff_t *offset) {
    const char *data = PSEUDO_FILE_CONTENT;
    size_t data_len = strlen(data);

    if (*offset >= data_len)
        return 0; // EOF

    if (len > data_len - *offset)
        len = data_len - *offset;

    if (copy_to_user(buf, data + *offset, len))
        return -EFAULT;

    *offset += len;
    return len;
}

static const struct file_operations pseudo_file_ops = {
    .read = pseudo_read,
};

// Directory operations
static struct dentry *pseudo_lookup(struct inode *parent_inode, struct dentry *child_dentry, unsigned int flags) {
    printk(KERN_INFO "pseudo_fs: Looking up %s in directory %s\n", child_dentry->d_name.name, parent_inode->i_sb->s_id);

    // If the requested file name is 'hello.txt', create it
    if (strcmp(child_dentry->d_name.name, PSEUDO_FILE_NAME) != 0)
        return NULL;  // Only handle 'hello.txt'

    struct inode *inode = new_inode(parent_inode->i_sb);
    if (!inode)
        return ERR_PTR(-ENOMEM);

    inode->i_ino = get_next_ino();
    inode->i_mode = S_IFREG | 0644;
    inode->i_op = NULL;
    inode->i_fop = &pseudo_file_ops;
    inode->i_atime = inode->i_mtime = current_time(inode);
    inode_set_ctime_to_ts(inode, current_time(inode));  // Set ctime

    d_add(child_dentry, inode);
    return NULL;
}

static struct inode_operations pseudo_dir_inode_ops = {
    .lookup = pseudo_lookup,
};

static struct file_operations pseudo_dir_ops = {
    .read = generic_read_dir,
};

// Superblock filling
static int pseudo_fill_super(struct super_block *sb, void *data, int silent) {
    printk(KERN_INFO "pseudo_fs: Filling superblock for pseudo_fs\n");

    // Create the root inode for the pseudo file system
    struct inode *root_inode = new_inode(sb);
    if (!root_inode)
        return -ENOMEM;

    root_inode->i_ino = get_next_ino();
    root_inode->i_mode = S_IFDIR | 0755;  // Directory with read-write-execute permissions
    root_inode->i_op = &pseudo_dir_inode_ops;
    root_inode->i_fop = &pseudo_dir_ops;
    root_inode->i_atime = root_inode->i_mtime = current_time(root_inode);
    inode_set_ctime_to_ts(root_inode, current_time(root_inode));

    // Set up the superblock and root
    sb->s_magic = PSEUDOFS_MAGIC;
    sb->s_root = d_make_root(root_inode);
    if (!sb->s_root)
        return -ENOMEM;

    // Now, create 'hello.txt' in the root directory
    struct dentry *hello_dentry = d_alloc_name(sb->s_root, PSEUDO_FILE_NAME);
    if (!hello_dentry)
        return -ENOMEM;

    struct inode *file_inode = new_inode(sb);
    if (!file_inode)
        return -ENOMEM;

    file_inode->i_ino = get_next_ino();
    file_inode->i_mode = S_IFREG | 0644;  // Regular file with read-write permissions
    file_inode->i_op = NULL;
    file_inode->i_fop = &pseudo_file_ops;
    file_inode->i_atime = file_inode->i_mtime = current_time(file_inode);
    inode_set_ctime_to_ts(file_inode, current_time(file_inode)); // Update ctime

    // Add the file to the root directory
    d_add(hello_dentry, file_inode);

    return 0;

}

static struct dentry *pseudo_mount(struct file_system_type *fs_type, int flags,
                                   const char *dev_name, void *data) {
    return mount_nodev(fs_type, flags, data, pseudo_fill_super);
}

static void pseudo_kill_superblock(struct super_block *sb) {
    kill_litter_super(sb);
}

static struct file_system_type pseudo_fs_type = {
    .owner = THIS_MODULE,
    .name = "pseudo_fs",
    .mount = pseudo_mount,
    .kill_sb = pseudo_kill_superblock,
};

// Module initialization
static int __init pseudo_fs_init(void) {
    return register_filesystem(&pseudo_fs_type);
}

// Module cleanup
static void __exit pseudo_fs_exit(void) {
    unregister_filesystem(&pseudo_fs_type);
}

module_init(pseudo_fs_init);
module_exit(pseudo_fs_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("Simple Pseudo File System Example");

```


# cat /proc/sys/kernel/tainted





