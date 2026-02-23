---
title: "SquashFS Direct I/O: Decompressing Without Blowing Up Your Page Cache"
draft: false
date: 2024-02-04
tags:
  - Linux
---

## Background

In our infrastructure, users' code files are distributed as squashfs images. Before execution, these images need to be decompressed onto disk. The problem? Standard decompression fills the page cache with data that's only read once, putting unnecessary memory pressure on the host — especially when pre-downloading files across many containers.

Our goal was clear: **decompress squashfs images without polluting the page cache.** To achieve this, we modified the [unsquashfs](https://github.com/plougher/squashfs-tools) tool to write decompressed data using direct I/O, which bypasses the kernel's page cache entirely. What seemed like a one-line flag change turned into a journey through alignment constraints, filesystem quirks, and sparse file edge cases. This post documents the pitfalls we hit along the way.

## How unsquashfs Works

Before diving into the changes, it helps to understand `unsquashfs` internals. The tool uses a producer–consumer threading model:

- **Producer threads** read and decompress data blocks from the squashfs image.
- **Consumer (writer) threads** receive these blocks and write them to disk.

Each block has a fixed size defined by `SQUASHFS_FILE_SIZE` (128 KB by default):

```c
/* default size of data blocks */
#define SQUASHFS_FILE_SIZE 131072
```

A single block may contain data for multiple files, but all blocks belonging to a given file arrive in order. The writer thread processes them sequentially:

```c
// unsquashfs.c (simplified)
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

	// Blocks for a file are ordered — write them sequentially
	for(i = 0; i < file->blocks; i++, cur_blocks ++) {
		// write block->buffer->data + block->offset, block->size
	}
}
```

At first glance, the modification seems straightforward: just open files with `O_DIRECT` and write the blocks out. But direct I/O comes with strict rules — read on.

## Implementing Direct I/O Writes

Direct I/O imposes two key alignment requirements:

1. **Buffer alignment** — the memory address of the write buffer must be aligned to the filesystem's block size.
2. **Write size alignment** — the number of bytes written must be a multiple of the disk's logical block size (typically 512 bytes).

We allocate an aligned buffer `dio_buf` at startup:

```c
posix_memalign((void **)&dio_buf, fs_block_size, SQUASHFS_FILE_SIZE);
```

The write logic then branches based on the data chunk size:

1. **Full block (size == `SQUASHFS_FILE_SIZE`):** The squashfs decompression buffer is already properly aligned, so we write directly from it — no copy needed.
2. **Partial block (size < `SQUASHFS_FILE_SIZE`):** We copy the data into `dio_buf` to guarantee pointer alignment, then round the write size up to the nearest multiple of `fs_block_size`.

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
		write_size = align_size * fs_block_size;
	}
	int res = write(file_fd, buf, write_size);
	// ...
}
```

## Handling Unaligned File Tails

The alignment requirement creates an unavoidable problem: almost no real file has a size that's an exact multiple of the block size. The last chunk of data will be smaller, and after rounding up, we end up writing a few extra padding bytes to disk.

The fix is simple in principle — after writing, call `truncate` to trim the file back to its true size.

### The XFS `truncate` Performance Trap

This worked perfectly on ext4, where decompression completed in seconds. But on **XFS**, we hit a showstopper: `truncate` on direct-I/O-written files was **orders of magnitude slower** — taking *minutes* instead of seconds. The root cause remains unclear (possibly related to XFS's extent management or speculative preallocation behavior).

Our workaround: for the final unaligned chunk only, **switch back to buffered I/O** using `fcntl` to remove the `O_DIRECT` flag:

```c
memcpy(dio_buf, cur, to_write);
if (fcntl(file_fd, F_SETFL, O_CREAT | O_WRONLY) == -1) {
	// error handling...
}
res = write(file_fd, dio_buf, to_write);
```

Since this final buffered write does generate page cache, we immediately flush it and evict the cached pages:

```c
fdatasync(file_fd);
posix_fadvise(file_fd, 0, file->file_size, POSIX_FADV_DONTNEED);
```

With this approach, XFS performance matched ext4 — decompression completed in seconds with negligible page cache overhead.

## Handling Sparse Files

Sparse files (files with "holes") are another edge case. For these files, there's no actual data to write — the decompressor only advances the file offset to create gaps. Adding direct I/O alignment logic on top of hole-punching and `lseek` operations would introduce significant complexity for little benefit.

For sparse files, we took the pragmatic route: use standard buffered I/O and then flush the cache afterward with `fdatasync` + `posix_fadvise`, same as the XFS workaround above.

## Summary

What started as "just add `O_DIRECT` to the write call" turned into a surprisingly involved patch. Here's a recap of the key lessons:

| Challenge | Solution |
|---|---|
| Buffer alignment for direct I/O | `posix_memalign` with filesystem block size |
| Write size must be block-aligned | Round up and pad the final chunk |
| Padding produces oversized files | `truncate` to correct size after writing |
| XFS `truncate` is extremely slow | Fall back to buffered I/O for the last chunk |
| Buffered I/O pollutes page cache | `fdatasync` + `posix_fadvise(DONTNEED)` to evict |
| Sparse files add too much complexity | Use buffered I/O + cache eviction |

Hopefully this saves someone else a few days of debugging. Direct I/O is powerful but unforgiving — the devil is very much in the details.