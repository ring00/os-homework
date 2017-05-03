# 实验八：文件系统

## 练习一

### sfs_io_nolock函数的实现

在进行I/O操作时，要特别考虑非对齐的读写问题。由于这种情况只可能出现在第一个以及最后一个块上，只要注意进行判断即可

```c
size = (nblks != 0) ? (SFS_BLKSIZE - blkoff) : (endpos - offset);
......
size = endpos % SFS_BLKSIZE;
```

对于整块的读写，可以使用`sfs_bmap_load_nolock`函数，对于非对齐读写，可使用`sfs_buf_op`。依次读取第一个块，中间块和最后一个块即可。

### Unix的pipe机制

一个pipe实际上就是存在于内存中的文件，对这个文件的操作要通过两个已经打开文件进行，它们分别代表管道的两端。pipe是一种独立的文件系统，有自己的数据结构，为了方便起见，我们可以在内部用一个FIFO队列实现其功能，当然，也可使用circular buffer实现。

一个file结构体如下，容易发现我们入手实现pipe的地方只有inode

```c
struct file {
    enum {
        FD_NONE, FD_INIT, FD_OPENED, FD_CLOSED,
    } status;
    bool readable;
    bool writable;
    int fd;
    off_t pos;
    struct inode *node;
    int open_count;
};
```

而一个inode结构体如下

```c
struct inode {
    union {
        struct device __device_info;
        struct sfs_inode __sfs_inode_info;
    } in_info;
    enum {
        inode_type_device_info = 0x1234,
        inode_type_sfs_inode_info,
    } in_type;
    int ref_count;
    int open_count;
    struct fs *in_fs;
    const struct inode_ops *in_ops;
};
```

其中inode_ops包含了一系列函数指针用于inode的操作，我们需要在VFS之下为pipe单独实现一个文件系统

首先修改inode，增加pipe_inode结构体

```c
struct pipe_inode {
    int buf_size;
    int pos;
    char *pipe_buf;
};

struct inode {
    union {
        struct device __device_info;
        struct sfs_inode __sfs_inode_info;
        struct pipe_inode __pipe_inode_info;
    } in_info;
    enum {
        inode_type_device_info = 0x1234,
        inode_type_sfs_inode_info,
    } in_type;
    int ref_count;
    int open_count;
    struct fs *in_fs;
    const struct inode_ops *in_ops;
};
```

然后几个主要操作大致实现如下

```c
int pipe_open(struct inode *node, uint32_t open_flags) {
    static const int kBUFSIZE = 4096;
    struct pipe_inode *pipe = vop_info(node, pipe_inode);
    pipe->buf_size = kBUFSIZE;
    pipe->pos = 0;
    pipe->pipe_buf = (char *)kmalloc(kBUFSIZE);
}

int pipe_close(struct inode *node) {
    struct pipe_inode *pipe = vop_info(node, pipe_inode);
    pipe->buf_size = 0;
    pipe->pos = 0;
    kfree(pipe->pipe_buf)
}

int pipe_read(struct inode *node, struct iobuf *iob) {
    struct pipe_inode *pipe = vop_info(node, pipe_inode);
    size_t transferred;
    iobuf_move(iob, pipe->pipe_buf + iob->offset, iob->io_resid, 1,
               &transferred);
    pipe->pos += transferred;
}

int pipe_write(struct inode *node, struct iobuf *iob) {
    struct pipe_inode *pipe = vop_info(node, pipe_inode);
    size_t transferred;
    iobuf_move(iob, pipe->pipe_buf + iob->offset, iob->io_resid, 0,
               &transferred);
    pipe->pos += transferred;
}
```

## 练习二

### Unix的硬链接和软链接机制

创建一个hard link实际上是新建一个文件结构然后指向已有的数据即可

* 复制`struct file`(由于指向同一块数据，需要浅复制)
* 将inode的引用计数加1
* 修改文件名称为用户指定的名称

一个soft link应当存储目标文件的地址，因此可以看成存有文件地址的文件

* 新建一个指向一个link的文件，即将目标文件地址保存为文件
* 当用户打开soft link时，先取出实际文件的路径，然后打开该路径
* 该路径也可能是另一个soft link，要反复进行到打开一个hard link为止

## 总结

### 实现与参考答案的区别

`load_icode`的主体基本一致，将argc和argv压栈的实现略微不同

### 知识点

* 文件系统抽象层VFS
* Simple FS文件系统
* 设备层文件IO层