---
title: squashfs 文件支持 directio 解压
draft: false
date: 2024-02-04
tags:
  - Linux
  - zh
---

做这个事情的出发点非常复杂，主要是用户的代码文件是 squashfs 格式，决定解压后使用，并且希望预下载的文件不要提高内存的压力。

所以最后决定将 [unsquashfs](https://github.com/plougher/squashfs-tools)这个工具改成 directio 写的模型，不产生 page cache。在实现 directio 的过程中，遇到一些坑，当时在网上也没找到什么解决办法，这里记录一下。

## unsquash 工具

squashfs 压缩的格式比较多，unsquashfs 在工作时，会有生产者（解压/read）和消费者（writer）线程。生产者会以定长大小SQUASHFS_FILE_SIZE，吐出一个 block 的数据。block 里可能包含多个文件的 buffer，但是每个文件的所有 buffer，在编程的时候是连续的，squashfs-tools 会处理好。

```c
/* default size of data blocks */

#define SQUASHFS_FILE_SIZE 131072
```

```c
// unsquashfs.c

while(1) {
	struct squashfs_file *file = queue_get(to_writer);
	
	int file_fd;
	// ...

	if(file == NULL) {
		queue_put(from_writer, (void *) exit_code);
		continue;
	} else if(file->fd == -1) {
	/* write attributes for directory file->pathname */
	}

	//  按文件拿到的 block 是有序的，逐个写入磁盘
	for(i = 0; i < file->blocks; i++, cur_blocks ++) {
	// do write
	// 通过block->buffer->data + block->offset, block->size 访问数据
	}
}

```

所以我们的思路很简单，把写入过程改成 directio 写就好了？额外再处理一下跨 block 的大文件。别着急，继续往下看。
## direct io 写

directio 写需要做 buffer 对齐，首先将 dio_buf 初始化为与 block 相同的 size。
```c
posix_memalign((void **)&dio_buf, fs_block_size, SQUASHFS_FILE_SIZE);
```

写入的时候，数据长度必须是磁盘逻辑块大小的整数倍，一般是 512 字节的整数倍。并且buffer 指针也必须是对齐。比如说，你不能用 dio_buf+3 作为 src。

所以这里做两种处理：
一：如果数据大小与 dio_buf 相同，说明是跨 block 的文件数据。直接复用 squashfs 的 buffer 用于写入数据，省去拷贝步骤。
二：如果数据块小于 dio_buf 大小。将数据块拷贝至 dio_buf（这是为了对齐待写入的buffer指针）
然后再将 dio_buf 用 directio 写入文件。

```c
int write_by_directio(int file_fd, char *buffer, int size, char *dio_buf) {
if (size == SQUASHFS_FILE_SIZE) {
	buf = buffer;
	write_size = size;
} else {
	buf = dio_buf;
	memcpy(dio_buf, buffer, size);
	int align_size = size / fs_block_size;
	if (size % fs_block_size != 0)
		align_size += 1;
	write_size = align_size*fs_block_size;
}
int res = write(file_fd, buf, write_size);
// ...
}
```

## 处理未对齐的数据

写入必须对齐后写入，但是现实中，绝大部分文件的大小都是不对齐的。剩下最后那部分数据，需要特殊处理。

这里的做法是，最后一个块会额外写数据。最后通过 truncate 的方式，将文件内容调整为正确大小。

### 特殊处理xfs文件系统

实际测试的时候，在 ext4 文件系统上，解压速度很快，秒级。但是在 xfs 文件系统上，truncate 却非常慢，到了分钟级，不太清楚为什么。
所以针对 xfs 文件系统，通过fcntl切换回 cache io 的方式来写入最后非对齐的数据。
```c
memcpy(dio_buf, cur, to_write);
if (fcntl(file_fd, F_SETFL, O_CREAT | O_WRONLY) == -1) {
// ...
}
res = write(file_fd, dio_buf, to_write);
```

切换回 cache io，那么最后一部分会产生 cache。通过 fdatasync+fadvise 将内存“挤出去”
```c
fdatasync(file_fd);
posix_fadvise(file_fd, 0, file->file_size, POSIX_FADV_DONTNEED);
```

这种方案，实测下来，性能跟 ext4 上差不多。也是秒级。

## 处理空洞文件

以上的处理，都在处理 buffer。如果是空洞文件，buffer 是空的，只会将文件指针跳过。各种 error case 处理起来太复杂。所以这种文件，直接按原来的方式，通过 cache io 写，然后挤掉内存就好。

## 总结

一开始以为只要在写入的地方加个 flag 就好，没想到这么多坑。记录下来，希望对大家有帮助。