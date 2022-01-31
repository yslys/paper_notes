# File Systems

### Lecture 1 - Intro
Every time you create a program on a 32-bit system, OS is going to assign 2^32
address space to such program.

+ What is a file?
    + A file is a sequence of bytes (with order).
    + A file is persistent on disk (even if power goes off).

+ How do files being stored on a disk?
    + A disk is divided into blocks.
    + A file is located on >= 1 disk blocks, not necessarily consecutive blocks.
    + E.g. a file with block number: 4 - 7 - 2 - 10 - 12.

+ How to retrieve a file located on separate disk blocks?
    + Option 1: linked list
        + For each block, we save a pointer to the next block, by which the order will persist.
        + Cons: 
            + Disk access will be sequential: slow for random access.
            + We want disk block size to be a power of 2, and we want the size of the content stored on such disk block to be a power of 2. Since we need to store a pointer in this case, it is hard to satisfy both conditions.

+ How to organize a file?
    + ```inode``` - index node.
        + A data structure to hook all the disk block together for a file.
        + It supports random access, and makes sure everything is a power of 2.

### Lecture 2 - inode & UFS
+ What does inode do?
    + inode maps the logical file into the physical disk block.

+ How large is an inode?
    + 256 Bytes.

+ What is the core data structure within an inode structure?
    + (For FreeBSD OS) 15 pointers.
        + 12 direct pointers pointing to disk blocks, representing the first 12 disk blocks of the file represented by inode.
        + 1 single indirection pointer, pointing to a pointer block - a disk block that stores only pointers to disk blocks.
        + 1 double indirection pointer, pointing to a block of single indirection pointers.
        + 1 triple indirection pointer, pointing to a block of double indirection pointers.
    + The order of the pointers determines the order of the file - starting from the first 12 pointers, then single indirection pointer, then double, then triple.

+ Direct access to any block of a file:
    + If you want to access to a particular position of a file, you would use a syscall called ```fseek()```, which requires a param called ```offset``` - the offset from the beginning of the file you want to access.
    + Such ```offset``` is divided by block size to determine which block will be accessed.

+ What happens when you ```open()``` a file?
    + 1st disk access:
        + The inode of the corresponding file will be cached from the hard disk to memory.
    + What happens when you ```fseek()``` with an offset?
        + If the offset lays in the first 12 direct pointers, then simply do another disk access to get the disk block.
        + If the offset lays in the single indirection pointer, then there will be 1 disk access to get the pointer block, and then another disk access for the data we are interested in.
        + If the offset lays in the double indirection pointer, then there will be two disk accesses to get the pointer block that contains the pointer to the disk block we want, and then another disk access.
        + If the offset lays in the triple indirection pointer, then (3 + 1) disk accesses.


+ Examples: Suppose each disk block is 4KB
    + If a file is less than 48KB, then using direct pointers would suffice.
    + If it is 64-bit architecture, then each pointer will be 8 bytes; then for each disk block, there will be 4KB/8 = 512 pointers. 

+ Extra info:
    + UFS2 (Unix FileSystem 2): 64-bit architecture, each inode is of size 256 bytes.
    + UFS1 (Unix FileSystem 1): 32-bit architecture, each inode is of size 128 bytes.

+ Something important to know about ```inode``` and file system:
    + inode is pretty efficient when handling small files.
    + inode also stores the order of the sequence of the bytes of the file.
    + For a file system, there are 5 different kinds of blocks (all stored on hard disk):
        + Super block
        + inode block
        + indirection (pointer) block
        + directory block
        + data block

Below is the structure of the ```ufs2_dinode```:
```
125 struct ufs2_dinode {
126     u_int16_t di_mode; /* 0: IFMT, permissions; see below. */
127     int16_t di_nlink; /* 2: File link count. */
128     u_int32_t di_uid; /* 4: File owner. */ 
129     u_int32_t di_gid; /* 8: File group. */ 
130     u_int32_t di_blksize; /* 12: Inode blocksize. */ 
131     u_int64_t di_size; /* 16: File byte count. */ 
132     u_int64_t di_blocks; /* 24: Bytes actually held. */ 
133     ufs_time_t di_atime; /* 32: Last access time. */ 
134     ufs_time_t di_mtime; /* 40: Last modified time. */ 
135     ufs_time_t di_ctime; /* 48: Last inode change time. */ 
136     ufs_time_t di_birthtime; /* 56: Inode creation time. */ 
137     int32_t di_mtimensec; /* 64: Last modified time. */ 
138     int32_t di_atimensec; /* 68: Last access time. */ 
139     int32_t di_ctimensec; /* 72: Last inode change time. */ 
140     int32_t di_birthnsec; /* 76: Inode creation time. */ 
141     int32_t di_gen; /* 80: Generation number. */ 
142     u_int32_t di_kernflags; /* 84: Kernel flags. */ 
143     u_int32_t di_flags; /* 88: Status flags (chflags). */ 
144     int32_t di_extsize; /* 92: External attributes block. */ 
145     ufs2_daddr_t di_extb[NXADDR];/* 96: External attributes block. */ 
146     ufs2_daddr_t di_db[NDADDR]; /* 112: Direct disk blocks. */ 
147     ufs2_daddr_t di_ib[NIADDR]; /* 208: Indirect disk blocks. */ 
148     int64_t di_spare[3]; /* 232: Reserved; currently unused */ 
149 }; 
```
+ The numbers in the comments: the offsets in bytes of the fields from the starting of the ```ufs2_dinode``` structure.
+ The **size of this inode structure is 256 bytes**.
+ How many inodes are there in one disk block (4KB)?
    + 4KB / 256bytes = 16 (inodes/disk block)
+ Which lines correspond to the direct blocks and indirect blocks?
    + Line 146 ```ufs2_daddr_t di_db[NDADDR];```: direct blocks.
        + ```NDADDR``` = (208-112)/8 = 12.
    + Line 147 ```ufs2_daddr_t di_ib[NIADDR];```: indirect blocks.
        + ```NIADDR``` = (232-208)/8 = 3.
+ Line 131 ```u_int64_t di_size;``` vs Line 132 ```u_int64_t di_blocks;```? 
    + This corresponds to **logical file size** and **physical file size**.
        + Logical file size: the size of the file.
        + Physical file size: the number of disk blocks to store this file. It not only contains the data block, but also the inode block, the indirection block.
        + See this program that seeks to a random position, and writes random bytes for 1000 times:
        ```
        #include <stdio.h>
        #include <stdlib.h>

        int main(void) {
            FILE *f1 = fopen("./sss.txt", "w");
            int i;

            for (i = 0; i < 1000; i++)
                {
                fseek(f1, rand(), SEEK_SET);
                fprintf(f1, "%d%d%d%d", rand(), rand(),
                        rand(), rand());
                if (i % 100 == 0) sleep(1);
                }
            fflush(f1);
        }
        ```
        + Logically, this file would be large, since for each seek & write, there will be a block associated to it. But the file will have lots of holes in it.
        + After compiling the code, if we execute ```$ du``` (disk usage), the output on my side is 28, meaning that there are 28 disk blocks for the current directory.
        + After running the program, execute ```$ du``` again, it shows 4076, meaning that 4076 disk blocks for this directory - subtracting the original 28, we get 4048 (~4KB) disk blocks for the newly written file.
        + How to determine the size of each **disk block**?
            + By searching on the internet, there are two ways:
                + w/ sudo: ```$ sudo tune2fs -l /dev/sda1 | grep -i 'block size'```
                + w/o sudo: ```$ stat -fc %s .```
            + In my case, I got 4096 bytes per disk block.
        + Therefore, the total file size on disk is ```~4KB disk blocks * 4KB/disk block```, which is 16MB.
        + The logical file size of the newly written file is - (by executing ```$ ls -al```) 2146220982 bytes (~2.1GB).
        + **Physical file size: ~16MB, and logical file size: ~2.1GB.**
    + Hence, in the above case, ```di_blocks``` is 4048 (bytes of blocks), ```di_blocks``` is 2146220982 (bytes).
    + Interesting applications:
        + When user opens a file (say, it's a movie with partially downloaded using bittorrent) and seek to a particular offset, OS will handle the rest. If for that offset, the file has not been downloaded, what OS will do is - instead of returning an error to the user, it will redirect that offset to the bittorrent client, and then bittorrent client will receive such request, and put such request at a higher priority to grab that piece, and send it back to OS. However, there will be some delay for the user to be able to watch that offset of the movie.
+ ```u_int64_t di_size;``` compared with ```u_int64_t di_blocks;```?
    + Can be equal, or greater than, or less than.
    + di_size is mostly less than di_block*blocksize

+ A nice property of the inode:
    + It is possible for us to use an inode represent a logical file.


+ **Slide 45**: A typical hard disk configuration, a.k.a file system. 
    + On top: a hard disk consists of several partitions.
    + In the middle: the file system associated with each partition.
        + The first block: boot block.
        + The second block: super block.
        + 3rd piece: the list of inode blocks.
        + 4th piece: directory blocks, data blocks and indirection blocks(within inodes).
    + On the bottom: the list of inode blocks.


+ How to recover a file?
    + **inode block**: the disk block that contains only the inodes. (4096 bytes) / (256 bytes/inode) = 16 inodes in each inode block.
    + What happens when removing a file:
        + Remove the entry to the inode from the directory.
        + The inode structure itself will not be replaced until you create a new file that requires an inode, which means, if your disk space is not full, then possibly your old inode is still there. Hence, as long as the old file has not been overwritten by new files, and as long as I can find the older version of the directory block that still has the inode number corresponding to the deleted file, then I will be able to access to every data block or indirect block of the old file.





### Lecture 3 - inode & directory & snapshot
File systems have to be:
+ Hierarchical (directory)
+ Efficient in time and space (inode)
+ Robust against all failures (soft update, fsck, snapshots - the three must work together)
+ Extensibility to new functionalities (vnode)

Below is the structure of the ```ufs2_dinode```:
```
125 struct ufs2_dinode {
126     u_int16_t di_mode; /* 0: IFMT, permissions; see below. */
127     int16_t di_nlink; /* 2: File link count. */
128     u_int32_t di_uid; /* 4: File owner. */ 
129     u_int32_t di_gid; /* 8: File group. */ 
130     u_int32_t di_blksize; /* 12: Inode blocksize. */ 
131     u_int64_t di_size; /* 16: File byte count. */ 
132     u_int64_t di_blocks; /* 24: Bytes actually held. */ 
133     ufs_time_t di_atime; /* 32: Last access time. */ 
134     ufs_time_t di_mtime; /* 40: Last modified time. */ 
135     ufs_time_t di_ctime; /* 48: Last inode change time. */ 
136     ufs_time_t di_birthtime; /* 56: Inode creation time. */ 
137     int32_t di_mtimensec; /* 64: Last modified time. */ 
138     int32_t di_atimensec; /* 68: Last access time. */ 
139     int32_t di_ctimensec; /* 72: Last inode change time. */ 
140     int32_t di_birthnsec; /* 76: Inode creation time. */ 
141     int32_t di_gen; /* 80: Generation number. */ 
142     u_int32_t di_kernflags; /* 84: Kernel flags. */ 
143     u_int32_t di_flags; /* 88: Status flags (chflags). */ 
144     int32_t di_extsize; /* 92: External attributes block. */ 
145     ufs2_daddr_t di_extb[NXADDR];/* 96: External attributes block. */ 
146     ufs2_daddr_t di_db[NDADDR]; /* 112: Direct disk blocks. */ 
147     ufs2_daddr_t di_ib[NIADDR]; /* 208: Indirect disk blocks. */ 
148     int64_t di_spare[3]; /* 232: Reserved; currently unused */ 
149 }; 
```

+ Are the file names stored in ```inode```?
    + As we can see from the above inode structure, it does not contain the file name. Hence, the **filename is not the part of the file itself**. This is because a file is just a sequence of bytes stored (sequentailly or discretely) on the hard disk, and inode only points to those bytes, with some extra information shown below:
        + ```u_int16_t di_mode; /* 0: IFMT, permissions; see below. */```: access mode of this file.
        + ```u_int32_t di_uid; /* 4: File owner. */```: who created this file.
        + ```u_int32_t di_gid; /* 8: File group. */```: who can access this file.
        + Time stamps.
        + ```u_int32_t di_kernflags; /* 84: Kernel flags. */```: flags associated with this file.

+ Where are the file names stored?
    + On **Slide 45**, it shows the typical hard disk configuration, a.k.a file system. In the middle layer, we can notice directory blocks. Those blocks contain a lot of pointers to the file names and inodes.

+ Therefore, there must be a data structure that stores the file names, and that is called **Directory entries```.
```
struct dirent {
	ino_t d_ino; /* inode number */
	char  d_name[NAME_MAX+1]; /* the name of the file */
};
```

+ How are ```dirent``` structured organized and where are they stored?
    + All directory entries are stored in directory blocks.
    + In each directory block, it has many ```dirent``` structures.
    + In each ```dirent``` structure, there is an inode number, and the file name.
    + Then file system can use the ```d_ino``` **inode number to retrieve the actual inode** of the file. 

+ What is the order of execution when we search for a file, e.g. ```/dir/filename```?
    + We need to first go to inode #2 to get the inode of the root. 
        + (inode #2 is always corresponding to the root, so whenever we want to search for a file in a file system, we always start with inode #2. inode is the first block of the inode block list, right after the boot block and the super block).
    + 1.1) Use the inode to fetch the corresponding directory block from disk.
    + 1.2) Inside the directory block, find the corresponding ```dirent``` structure with ```dirent->d_name``` being "dir".
    + 1.3) Inside that ```dirent``` structure, it has ```dirent->d_ino``` - the inode number. Use that inode number to retrieve the corresponding inode structure.
    + 2.1) Use such inode to fetch the corresponding directory block (with name being "dir").
    + 2.2) Inside the directory block, find the corresponding ```dirent``` structure with ```dirent->d_name``` being "filename".
    + 2.3) Inside that ```dirent``` structure, it has ```dirent->d_ino``` - the inode number. Use that inode number to retrieve the corresponding inode structure.
    + 3) Now, we have the inode for the file ```/dir/filename```. Then we can access the actual content depending on the size of the file and the offset user passed in.


+ Example on slide 47 and 48:
    + Slide 47: a graph that represents a directory hiererachy starting from root.
    + Slide 48: 
        + left - the inode structures
        + right - directory blocks
            + 1st column: the name of the file in each dirent structure
            + 2nd column: the inode of the file in each dirent structure

+ **Hard link: two directory entries with different names, but the same inode number.**
    + This allows us to have multiple names of the same file.
    + This corresponds to ```ex``` and ```vi``` in the example.
+ ```int16_t di_nlink; /* 2: File link count. */``` - the **hard link**.
    + Now it is time to revise this field in the inode - it stores the # of directory entries associated with this inode number.

+ What happens if the hard link becomes 0?
    + It means of all the directory entries in all the directory blocks, none of them is associated with this inode anymore.
    + It means we can remove that file.

+ What does file system do when we remove that file?
    + inode block needs to be modified
    + directory block needs to be modified
    + free block bitmap needs to be modified
    + free inode bitmap needs to be modified

+ What if system crashes in the middle of removing a file (as mentioned above)?
    + UFS can guarantee that the file system is consistent regardless of how the system crashes.
    + UFS achieves it by three things -
        + soft update
        + snapshot
        + background FSCK


#### Snapshot of the FS
+ What is a snapshot? (Slide 67, Slide 84)
    + A snapshot is actually a **file** located on disk. There is an **inode** associated to it.
    + A snapshot stores - for each disk block, whether such block is 
        + 1) not used (1), or 
        + 2) not yet copied (0) or
        + 3) the pointer to the copied blocks.
    + A snapshot stores what the FS (disk blocks) was like when the last time a snapshot was took.

+ What will happen if user modified something in the FS (Slide 63)?
    + After you modify something in your FS, there will be inconsistency between what FS currently is (figure on the left), and the previous snapshot FS took (figure on the right). Such inconsistency is the red part on the figure.
    + FS will not take a snapshot as soon as user has modified a single file; instead, FS will take a snapshot in a **Copy-on-Write** manner - a snapshot will not be taken until the modified file will be overwritten.
    + With an active file system, if 99% of your files are not modified, then FS will not do any copy on those 99% of the files.

+ What happens when FS is taking a snapshot (from a higher level perspective)?
    + Freeze all activities related to FS.
    + Copy everything to "somewhere else".
    + Resume the activities.

+ How long will it take to take a snapshot for a FS with 2TB?
    + A few seconds.

+ Why could a snapshot can be taken in such short period of time?
    + **Copy-on-Write**: try to avoid writing to disk until the data is about to be overwritten. It is a lazy way of writing to disk (doing the copy).
    + FS will take a snapshot in the **Copy-on-Write** manner. In other words, with an active file system, if 99% of your files are not modified, then FS will not do any copy on those 99% of the files.
    + While taking a snapshot, FS only need to store simple info: whether a disk block is not used or not yet copied, or the pointer to the copied block. If a disk block has been copied already, then no need to take a snapshot on that disk block. If a disk block has not been copied since last time FS took a snapshot, then at the time when the disk block is to be overwritten, a snapshot must be taken before being overwritten.
    + That is why not all the disk block need to be "copied", which makes taking a snapshot much faster.

+ **How does FS take a snapshot?** (slide 84)
    + Go through **free blocks bitmap** (0: not free, 1: free), and for each bit in the bitmap, if it is free, then in the corresponding direct disk blocks (```di_db[NDADDR]```) or indirect disk blocks (```di_ib[NIADDR]```) in the (snapshot) inode structure, we will set it to be ```not used```. If the bit in the bitmap is not free, then we need to set the corresponding part in the (snapshot) inode structure to be ```not yet copied```.

+ **How will snapshot be changed when user wants to modify n disk blocks?** (slides 85, 86)
    + Note: a snapshot already exists. When user modifies a file in user space, it is not necessarily the disk block will be modified. Only when the FS actually tries to write to the disk, snapshot occurs. Hence, the following steps are done before actually modifying the disk blocks.
    + **Only the very first snapshot (i.e. no other older snapshots) is done by going through free block bitmap, and mark those to be "not used" or "not yet copied" one by one. For later snapshots, we need both the free block bitmap and the older snapshot file - and the snapshot file will update.**
    + 1) For the to-be-modified blocks that are "not used" in free blocks bitmap, then simply update the corresponding blocks within the snapshot inode to be "not copied". Done.
    + 2) For the disk blocks (say, n) that are "not copied", go through free blocks bitmap to find n free blocks, and make a copy of the n disk blocks to be modified.
    + 3) After making a copy, the direct disk blocks (```di_db[NDADDR]```) or indirect disk blocks (```di_ib[NIADDR]```) in the (snapshot) inode structure will NO longer store "not used" or "not yet copied", but store the pointer to the newly copied disk block.
    + **Followup question**: now, the pointers within snapshot inode contain pointers to the copied disk blocks. If The user tries to modify again, what will happen? Does it force FS to write to disk block? Or FS creates another new snapshot which treats the newer old file as the old file?
        + It will notice that the snapshot inode has the corresponding pointer being not yet copied, and will not write to that inode pointer. So everytime we want to recover to a snapshot inode, we can only recover to what it was when we first took the snapshot.
    + **Followup question**: What if two snapshots choosing the same disk block to store the copy?
        + That is absolutely possible, and in that way, we can save a lot of space.


+ Since we are using exactly 1 inode to store the snapshot, will the number of disk blocks exceed the max pointers an inode could handle?
    + No. When configuring the FS, it needs to make sure that max # pointers an inode can handle > # disk blocks.
    + Hence, for a very large partition, the disk block size might be 4KB or 16KB, etc.

+ Where is the snapshot kept?
    + Stored in the free blocks.

+ In a typical FreeBSD file system, there are 20 snapshot files allowed, which means, every time user tries to make a modification to the disk blocks that are "not yet copied", FS needs to check all those 20 snapshots to see if any of the snapshot file needs to do a copy (i.e. write to disk). Why?
    + In order to answer this question, we first need to understand what snapshots are used for. Snapshots stores various versions of what the disk partition was like at the time such snapshot was taken. Given a snapshot, we are able to recover the disk partition to be exactly what is described by the snapshot.
    + Hence, no matter what application tries to modify the disk block, such modification will be stored by every snapshot, so that for each snapshot, it stores this particular modification so that FS of this disk partition will be able to recover to any of the snapshots given this single modification.

So we save some time everytime when we create a new snapshot, but we pay a bit more when we actively modify the disk block.

If we take a snapshot, the next snapshot will remember the previous snapshot.


Example of creating a snapshot:
```
$ mkdir /backups/usr/noon

## -o snapshot: the option passed to mount command indicating creating a snapshot
## create a snapshot for the file /sur, and the snapshot is /usr/snap.noon
$ mount –u –o snapshot /usr/snap.noon /usr

## we can then use mdconfig to look at the snapshot
$ mdconfig –a –t vnode –u 0 –f /usr/snap.noon
$ mount –r /dev/md0 /backups/usr/noon

/* do whatever you want to test it */

$ umount /backups/usr/noon
$ mdconfig –d –u 0
$ rm –f /usr/snap.noon

```
In the example on Slide 73 - 79: when we create a snapshot /usr/snap.noon, the logical file size of the snapshot is 16GB and the number of physical disk blocks are 9136 bytes (by executing ```$ du```). Note that for every snapshot file, the # of physical disk blocks are roughly the same. 

After that, we do a make clean, to remove lots of files. Then, we check the size of the snapshot again, the logical size does not change, but the # of physical disk blocks are much larger - 97920 bytes. This is because when we removing those files by make clean, lots of the corresponding disk blocks to be removed are marked "not yet copied" in the snapshot inode. Hence, we need to make lots of copies so that we could recover to what it was before make clean.


+ Where is the snapshot source code?
    + ```/usr/src/sys/ufs/ffs/ffs_snapshot.c```
    + It is a kernel service.
    + Under the ```usr/src/sys/ufs``` directory, there are two directories: ffs and ufs. ffs contains files regarding to **inode**, and services (like snapshot) related to inode will be placed under ffs directory. ufs contains the files for **directory (dirent)** and other related services.

### Lecture 4 - background FSCK (FS consistency check), and soft update
When a FS crashed, we can recover it by taking the snapshot. But if the total time to recover (by merely using snapshot) takes quite a long time, we want to find a way to reduce it. That is why **background FSCK** is introduced so that while FS is recovering, users are allowed to use the FS immediately. Hence, **background FSCK** is to shorten the time for the FS to be available after a crash.

When FS crashed, and you want to use the machine again, after you turn the machine on, before user is able to use the FS, a snapshot is required to be taken. Then, background FSCK starts, while user is able to use FS at the same time.


#### FSCK (Note: NOT background FSCK)
+ Inconsistency problem:
    + In order to achieve higher performance in disk accesses, OS maintains a buffer cache so that disk blocks can be cached to memory, or even be cached to cache (one level higher than memory). In other words, there might be several copies of the same disk block: 
        + 1) on disk itself
        + 2) in memory
        + 3) in cache
    + So when user modifies a disk block, it first reveals in the disk block in cache, then the disk block in memory, but there must be inconsistency between the disk block in memory and on disk.
    + Such inconsistency problem will persist until we flush such disk block to disk.
    + Not only the disk block that contains data will cause inconsistency problems, but also those corresponding data structures like inode block, directory block:
        + inode contains a timestamp to be modified.
        + Some other correlated data structures needs to be modified as well.
    + Hence, whenever user tries to modify a single file, all the corresponding disk blocks (e.g. data block, inode block, directory block) in memory will be placed in the buffer cache, and all those blocks will be different from the disk blocks on disk.

+ How could we maintain FS consistency?
    + The core idea: **the ordering of updates from buffer cache to disk is critical**.
        + e.g. If the directory block is written back before inode and the system crashes, the directory structure will be inconsistent.
    + 1) Less efficient approach, and it still does not 100% guarantee consistency after crash: 
        + Use unbuffered I/O when writing inodes or pointer blocks.
        + Use buffered I/O for other writes and force sync every 30 seconds.
    + 2) Detect and Fix - what most FS does:
        + Detect the inconsistency.
        + Fix them according to the rules.
        + Running FSCK.

Below describes how **FSCK detects the inconsistency** in blocks and directory entries:

+ Two ways of consistency checks: block consistency and file consistency
    + Block consistency check (what FSCK does - really expensive, but **NOT background FSCK**)
        + There are two core data structures:
            + **Block-in-use table**: an array of integers
                + This table is only constructed during FSCK - a large amount of time.
                + 1:block is in-use, 0:block is not in-use.

            + **Free block bitmap**: FS needs to know: on hard disk which block is free to use
                + This bitmap is maintained all the time.
                + 1:free, 0:not free.
        + How to construct the **Block-in-use table**?
            + For a disk block, if there is an inode, in which there is a pointer block that stores the pointer to that particular disk block, then it means such disk block is in use.
            + Hence, we need to examine every pointer block in every single inode. 
                + If for a single disk block, if there is one a pointer block pointing to it, then such disk block will be incremented by 1. 
                + If none of the pointer blocks in all inodes points to a disk block, then such disk block should be marked as not in-use, or free (0).
            + **Ideally, if the FS is consistent, then the values in Block-in-use array should be 0 or 1, since there should be at most 1 pointer block in an inode pointing to an in-used block. There should not be another pointer pointing to the same block.** 
        + If FS is consistent, then the Block-in-use table should be equal to the reverse of the Free block bitmap (0->1, 1->0), for instance:
            + 0 1 1 1 0 0 0 1 0 (Block-in-use table)
            + 1 0 0 0 1 1 1 0 1 (Free block bitmap)
        + If FS is inconsistent, for instance, the last three bits:
            + 0 1 1 1 0 0 0 1 0 **0** **1** **2** (Block-in-use table)
            + 1 0 0 0 1 1 1 0 1 **0** **1** **0**(Free block bitmap)
        + Let's consider the above three pairs of bits ```(block-in-use table, free block bitmap)``` for inconsistency one by one:
            + (0, 0)
                + There is no pointer in inode pointing to that disk block but that disk block is not free.
                + This is not that danger since no inode is pointing to that disk block.
            + (1, 1): 
                + there is a pointer in inode pointing to that disk block but that disk block is free.
                + This is very danger, since that means there is an inode using that disk block, but from the OS's perspective, it will regard such disk block as free. Hence, that disk block might be assigned to other files later which invalidates the previous inode.
                + **Soft update**: prevents this case from happening but allows (0, 0) to happen.
            + (2, 0): there are two pointers in inode pointing to that disk block and that disk block is not free.

    + File consistency check (directory structure consistency check)
        + In each inode, it has ```di_nlink``` field to indicate how many hard links there are for this inode, or how many directory entries having the same inode number.
        + In this case, we are going to build another table similar to Block-in-use table, which is also an integer array, with each element of the array being the **#of directory entries pointing to this inode**.
            + Starting from the root of the FS (inode #2), go through every ```dirent``` structure to find each corresponding inode number, and increment by 1. So if there are n directory entries having the same inode number x, then for the corresponding position x in the array, it will have a value of n.
            + We refer to each element in this array as **D**.
        + We also need another table that stores **```di_nlink``` field of each inode**.
            + Traverse all the inodes, and put ```di_nlink``` field in the array.
            + We refer to each element in this array as **L**.
        + Comparison between **D** and **L**:
            + D == L: files are consistent.
            + D < L: inconsistent.
                + This is less dangerous - just wasting some disk space.
                + Consider the following simpler case:
                    + di_nlink == 2, but only 1 dirent pointing to this inode.
                    + When the dirent removes the file, di_nlink field in the inode will be 1. So all the disk blocks corresponding to this inode will still be there, and will not be marked as free in Free block bitmap until "Garbage Collection" is executed. 
            + D > L: inconsistent.
                + This is very dangerous.
                + Consider the following simpler case:
                    + di_nlink == 1, but 2 dirents pointing to this inode.
                    + If one of the two dirents tries to remove the file corresponding to this inode, then the di_nlink field will be 0, indicating no dirent is associated with this inode number, hence this inode is marked as free, which might be overwritten by other applications. But the dirent that did not remove the file will still consider that the file is still there, so when it tries to access the file, it may not be able to access the correct file as it was.

+ Metadata operations
    + What is metadata?
        + The data blocks are not the metadata. The inode, Free block bitmap, Free inode bitmap, directory entry, etc. are all metadata.
    + Metadata operations modify the structure of the FS.
        + Creating, deleting or renaming files, directories or special files.
    + Metadata is actually maintaining the integrity of the FS.
        + **If you lost all the metadata, you might lose the whole FS.**
        + It is ok to lose some of the data blocks since we can use metadata to recover the FS to what it was when taking a snapshot.

Hence, metadata is extremely important and we do not want to lose it. We need a way to guarantee the integrity of the metadata.

+ How to guarantee the integrity of the metadata?
    + FFS uses synchronous writes for metadata, i.e. every time metadata is modified, it must be written back to the disk immediately. 
    + These writes will be blocking, i.e. nothing else can be done while the metadata is being written to the disk.
    + But using synchronous writes will largely impact the performance. The cost of metadata updates will be high.
    + For data blocks, the writes might be cached and flushed later.

From the above, we know that blocking writes of the metadata will strongly impact the performance, can we find out a way that we use less blocking writes (synchronous writes) but still maintains metadata integrity? Yes, by using **Soft Updates**.

#### Snapshot, Soft update, and background FSCK
Before diving into Soft update, we first need to figure out the relationship between the three.

+ Snapshot: gets a consistent global view of the whole FS
+ Background FSCK: garbage collect the lost ones
    + Background FSCK is - take a snapshot of FS, then run background FSCK against the snapshot whenever OS schedules it to run.
    + Foreground FSCK is slow.
+ Soft update: prevents Block inconsisty with (1, 1), and D > L
    + This makes sure that FS is safe to use while background FSCK is running.
    + Although this does not prevent garbage generated, garbage collection will be handled by background FSCK.

#### Soft Updates
Soft updates allows us to perform "dedalyed write" on metadata to the disk. But for the cached metadata, we need to make sure to **follow a particular order of writing those cached metadata back to disk**.

On deleting a file:

+ When deleting a file, we are deleting two things in order to let deletion complete.
    + The copy of disk block in memory
    + The disk block in disk

+ When deleting a file, what metadata needs to be modified (i.e. write to disk)?
    + ```dirent``` - the directory entry corresponding to the file removed should be deleted.
    + ```inode``` - decrement di_nlink.
    + Free block bitmap - for the free disk block.
    + Free inode bitmap - for the free inode.
+ Suppose the above 4 data structures are located on 4 different disk blocks,
    + In memory, there will be 4 disk blocks (pages) that are dirty.
    + Then needs to write the dirty blocks to disk.

+ What is the order of the 4 writes so that in between the 4 writes, no matter when a crash happens we can guarantee that 1) no block inconsisty with (1, 1) and no D > L?
    + If I write the inode block to disk before writing the Free inode bitmap, and system crashes, then the corresponding part of the Free inode bitmap will be still 0 (not free), this is acceptable since garbage collection in background FSCK will be able to handle it.
    + If I write the inode block to disk before writing the directory block, and system crashes, then in the directory block, it will consider that the file still exists. However, the inode block has already been modified with ```di_nlink``` being 0. Although free inode bitmap has not updated, background FSCK will be able to notice it and mark such inode to be free. Then it will be very danger since if later such inode has been assigned to another file, the directory entry will never be able to get the old file.
    + The correct order is as follows:
        + 1) Directory block
        + 2) inode block
        + 3) 4) free block bit map, and free inode bitmap

On creating a file, the order is going to be reversed. The goal is still to prevent 1) Block inconsisty with (1, 1), and 2) D > L
+ 1) Write the inode bitmap
+ 2) Write the inode block
+ 3) Write the directory block

On renaming a file (deleting and creating), both the directory block and inode block will be modified. The order is:
+ 1) increment the inode di_nlink by 1 (+1)
+ 2) add a new name in dirent
+ 3) remove the old name in dirent
+ 4) decrement the inode di_nlink by 1 (-1)

If the file is in the same directory, then only modifying the dirent name would suffice, but if in a different directory, then needs to follow the above 4 steps.

**Rules for soft update**:
+ For creating a file: never point to a structure before it has been initialized. So the data blocks must be written to disk first, before we can start writing the inode to disk. Similarly, we must first write the inode to disk, then we can write the corresponding directory entry to disk.
+ For deleting a file: never reuse a resource before nullifying all previous pointers to it. So when we delete a file, we first need to write the directory entry to disk first, then update the inode structure in disk.
+ For renaming (deleting and creating) a file: never reset the old pointer to a live resource before the new pointer has been set.




### Lecture 5 - Circular Dependency in Soft Updates & Journaling
https://web.stanford.edu/~ouster/cgi-bin/cs140-winter13/lecture.php?topic=recovery

Consistency conditions:
+ L >= D
    + To avoid D > L
    + Deletion: first dirent then inode (order of writing to disk)
    + Creation: first inode then dirent
+ (block_in_use_table\[B\], free_block_bitmap\[B\]) cannot be (1, 1)
    + (1, 0), (0, 1) are normal
    + (0, 0) is acceptable - waste some space, needs garbage collection
    + Deletion: (1, 0) -> (0, 1) - i.e. (1, 0) ~ (0, 0) ~ (0, 1)
    + Creation: (0, 1) -> (1, 0) - i.e. (0, 1) ~ (0, 0) ~ (1, 0)

Dependency relationships:
+ Problem setup: two blocks dirty blocks (dir block and inode block), we delete file "foo" and create a new file "bar".
    + Inside dir block: foo, and "NEW bar"
    + Inside inode block: inode-foo, and inode-NEW-bar
    + The two blocks are now in memory, with modified values (already deleted foo and inode-foo, created NEW bar and inode-NEW-bar), but not yet written to disk.

+ What is the order to write to disk?
    + For deleting foo, the order should be: dirent block then inode block.
    + For creating bar, the order should be: inode block then dirent block.
    + So there will be a conflict since there is a circular dependency.

+ **How to solve the circular dependency issue**?
    + Whenever there is a loop, we need to break that loop. That is called "roll back".
    + Consider the example on Slide 144:
    + 1st row: where circular dependency occurs
        + inode #4 and dirent A are the newly created in memory.
        + inode #5 and dirent B have been deleted in memory.
    + In order to solve the circular dependency, we need to waste one more disk block write. What we need to do is to roll back one of the two operations (creation and deletion) that causes the conflict. Suppose we choose to roll back the creation.
    + 2nd row: roll back creation (only deletion is done)
        + Since we are doing deletion only, we first need to write directory block to disk. But before that, we need to make a copy of the directory block to store the dirent A.
        + We then write the directory block to disk, with dirent A being empty.
        + On the right of the 2nd row, the original dirent B has been removed, while dirent A is not written there.
    + Now, the circular dependency has been solved, and we need to do the creation. For creation, we need to first write the inode block, then the directory block, that is why in row 3, we are writing the inode block. Besides, for the deletion that we done previously, we have already written the directory block, so it is also the time to write the inode block. Hence, in this single write (inode block), we write both the inode we created and the inode we deleted.
    + Finally, on the 4th row, we need to copy what is was in dirent A to the directory block to be written, and then we write the directory block to disk, so that dirent A is written to disk.

+ In the above example, we roll back creation. Can we roll back deletion?
    + We always roll back creation. We do not roll back deletion.
    + Reason?
        + Before we roll back, we need to consider what we have in memory. Roll back creation (row 2) means we roll back the creation of the dirent A in disk. Since before creation, the corresponding dirent in disk is NULL, we can safely overwrite that part with an empty dirent.
        + However, if instead, we roll back deletion and do creation first, we first need to make a copy of inode #5, zero inode #5 in inode block, and then write only inode #4 to disk (overwrite the inode block in disk). However, that will also overwrite the inode #5 in disk. So we must go to disk first, make a copy of what inode #5 was, and then fill that old value to the inode block, then we can safely overwrite the inode block in disk. That requires another disk read.


#### Journaling
Core idea: **write-ahead logging**

+ Write-ahead logging ensures that the log is written to disk before any blocks containing data modified by the corresponding operations.
+ Either all or nothing: either all the dirty blocks are written to the hard disk, or none of them are written to the hard disk.
+ No need to worry about circular dependency, since all the corresponding blocks will be written at once.
+ In journaling, first write the dirty blocks to the log. Such log is a specific area in disk. Then, after those dirty blocks have been written to the log, the file system will then write the log (i.e. the dirty blocks) to where they should be written to.
+ When writing to the log, system crashes, then after rebooting the system, FS will notice that the log is half-completed, then nullify those blocks in log.
+ After completing the log, system crashes, then after reboothing, FS will notice that the log is completed, and write to disk from the beginning of the log.
+ Every block needs to be written twice.



+ Comparison between journaling and soft update in various application scenarios?
    + Journaling has a simpler design than soft updates, easier to maintain and develop.
    + If the application you are working on is important, and you do not want to lose all the data, then soft update would be better.
    + One advantage of soft update is "when you can start using the FS after a crash".
        + This is critical to real-time system, critical mission system.
        + Soft update: after the snapshot has been taken can user starts to use the FS. You can start using FS even without taking a snapshot (if we do not care about the wasted space caused by crash).
        + Journaling: if system crashes after the log has been completed, but not yet written to the corresponding parts of disk, then we need to wait until log has been fully written.


### Lecture 6 - vnode & virtual file system
+ The layers of file system in Linux (ordered from top to bottom)
    + VFS: vnode
    + UFS: dirent
    + FFS: inode

+ vnode is a layer of abstraction in file system, right above the UFS layer (dirent).
+ vnode provides flexibility that we can introduce new services. Hence, a lot of OS services today are implemented in the vnode layer.
+ Examples: 
    + virus scanning service
        + Using vnode
        + Adding a virtualization layer for checking whether there is some kind of malicious signature in the files.
    + crypto file system
        + Want to encrypt the files and store to hard disk.
        + Encryption and decryption are implemented in vnode.

Before diving into vnode, we first need to know where such structure is defined. The first structure we will see is ```struct proc```, the structure of a process.

+ ```sys/proc.h```
    + This header file contains the data structure to capture all the processes or threads in kernel.
    + We will look into ```struct proc``` - a process
    + 
    ```
        struct proc {
            LIST_ENTRY(proc) p_list;        /* (d) List of all processes. */
            TAILQ_HEAD(, ksegrp) p_ksegrps; /* (c)(kg_ksegrp) All KSEGs. */
            TAILQ_HEAD(, thread) p_threads; /* (j)(td_plist) Threads. (shortcut) */
            TAILQ_HEAD(, thread) p_suspended; /* (td_runq) Suspended threads. */
            struct ucred    *p_ucred;       /* (c) Process owner's identity. */
            struct filedesc *p_fd;          /* (b) Ptr to open files structure. */
            struct filedesc_to_leader *p_fdtol; /* (b) Ptr to tracking node */
                                            /* Accumulated stats for all threads? */
            struct pstats   *p_stats;       /* (b) Accounting/statistics (CPU). */
            struct plimit   *p_limit;       /* (c) Process limits. */
            struct vm_object *p_unused1;    /* (a) Former upages object */
            struct sigacts  *p_sigacts;     /* (x) Signal actions, state (CPU). */
    ```
    + ```TAILQ_HEAD(, ksegrp) p_ksegrps```: kse group - kernel schedulable entity group
        + A process can contain more than 1 threads.
        + When a thread is mapped to a hypervisor or hardware, it is assigned a KSE - kernel schedulable entity.
    + ```struct filedesc *p_fd```: pointer to open files structure
        + Represents all the files that current process has access to.

Now, let's take a look at the ```struct filedesc``` - the structure for file descriptor.

+ ```sys/filedesc.h```
    + 
    ```
        50 struct filedesc {
        51     struct  file **fd_ofiles; /* file structures for open files */
        52     char    *fd_ofileflags;         /* per-process open file flags */
        53     struct  vnode *fd_cdir;         /* current directory */
        54     struct  vnode *fd_rdir;         /* root directory */
        55     struct  vnode *fd_jdir;         /* jail root directory */
        56     int     fd_nfiles;              /* number of open files allocated */
        57     NDSLOTTYPE *fd_map;             /* bitmap of free fds */
        ...
    ```
    + ```struct file **fd_ofiles;```: this is the list of pointers to open files owned by this file descriptor structure.

Next, we need to go to check ```struct file```, which represents an open file.

+ ```sys/file.h```
    + 
    ```
        106 struct file {
        107     LIST_ENTRY(file) f_list;/* (fl) list of active files */
        108     short   f_type;         /* descriptor type */
        109     void    *f_data;        /* file descriptor specific data */
        110     u_int   f_flag;         /* see fcntl.h */
        111     struct mtx      *f_mtxp;        /* mutex to protect data */
        112     struct fileops *f_ops;  /* File operations */
        113     struct  ucred *f_cred;  /* credentials associated with descriptor */
        114     int     f_count;        /* (f) reference count */
        115     struct vnode *f_vnode;   /* NULL or applicable vnode */
        116 
        117     /* DFLAG_SEEKABLE specific fields */
        118     off_t   f_offset;
        ...
    ```
    + ```off_t f_offset;```: the offset of the start of the current file (```fseek()```).
    + ```struct vnode *f_vnode;```: this is the vnode we will be talking about.


```struct proc``` -> ```struct filedesc``` -> ```struct file``` -> ```struct vnode```.
+ ```sys/vnode.h```
    + 
    ```
        107 struct vnode {
        108     struct  mtx v_interlock;                /* lock for "i" things */
        109     u_long  v_iflag;                        /* i vnode flags (see below) */
        110     int     v_usecount;                     /* i ref count of users */
        111     long    v_numoutput;                    /* i writes in progress */
        112     struct thread *v_vxthread;              /* i thread owning VXLOCK */
        113     int     v_holdcnt;                      /* i page & buffer references */
        114     struct  buflists v_cleanblkhd;
        115     struct  buf      *v_cleanblkroot;
        116     int              v_cleanbufcnt;
        117     struct  buflists v_dirtyblkhd;
        118     struct  buf      *v_dirtyblkroot;
        119     int     v_dirtybufcnt;
        120     u_long  v_vflag;                        /* v vnode flags */
        121     int     v_writecount;                   /* v ref count of writers */
        122     struct  vm_object *v_object;
        123     daddr_t v_lastw;                        /* v last write (write cluster) */
        124     daddr_t v_cstart;                       /* v start block of cluster */
        125     daddr_t v_lasta;                        /* v last allocation (cluster) */
        126     int     v_clen;                         /* v length of current cluster */
        ...
    ```
    + Clean buffer & Dirty buffer: line 114 - 119
        + Clean buffer: the ```buf``` structure that reads in but not modified - the list of blocks accessed by ```read()```
        + Dirty buffer: the list of blocks modified.

+ Figure on slide 180
    + For each process, there is a structure ```vm_map``` (virtual memory map) associated to it, which represents every virtual memory piece to allocate for this current process.
    + For ```vm_map```, it contains a linked list of ```struct vm_map_entry```.
































How is rollback performed? nullify the value in memory and write to disk? or it is done in disk only?
If nullifying the value in memory and write to disk, then for creation, roll back will also cause the data in memory to be lost and we cannot recover if the data is not in cache. The only way is to make a copy somewhere else.


Questions:
+ Is the following question described correctly? Is it what was discussed in lecture?
+ **How will snapshot be changed when user wants to modify n disk blocks?** (slides 85, 86)
    + Note: a snapshot already exists. When user modifies a file in user space, it is not necessarily the disk block will be modified. Only when the FS actually tries to write to the disk, snapshot occurs. Hence, the following steps are done before actually modifying the disk blocks.
    + **Only the very first snapshot (i.e. no other older snapshots) is done by going through free block bitmap, and mark those to be "not used" or "not yet copied" one by one. For later snapshots, we need both the free block bitmap and the older snapshot file - and the snapshot file will update.**
    + 1) For the to-be-modified blocks that are "not used" in free blocks bitmap, then simply update the corresponding blocks within the snapshot inode to be "not copied". Done.
    + 2) For the disk blocks (say, n) that are "not copied", go through free blocks bitmap to find n free blocks, and make a copy of the n disk blocks to be modified.
    + 3) After making a copy, the direct disk blocks (```di_db[NDADDR]```) or indirect disk blocks (```di_ib[NIADDR]```) in the (snapshot) inode structure will NO longer store "not used" or "not yet copied", but store the pointer to the newly copied disk block.
    + **Followup question**: now, the pointers within snapshot inode contain pointers to the copied disk blocks. If The user tries to modify again, what will happen? Does it force FS to write to disk block? Or FS creates another new snapshot which treats the newer old file as the old file?
    + **Followup question**: What if two snapshots choosing the same disk block to store the copy?

+ Is the following saying correct?
> Note: a snapshot already exists. When user modifies a file in user space, it is not necessarily the disk block will be modified. Only when the FS actually tries to write to the disk, snapshot occurs. Hence, the following steps are done before actually modifying the disk blocks.


What will happen if no extra disk blocks are available?
+ **Followup question**: What if two snapshots choosing the same disk block to store the copy? I remember in the example where we do make clean, all the snapshots should copy the removed files. Hence, I think the answer is probably - every snapshot should be updated sequentially, or when accessing the free block bitmap, needs to acquire a mutex. What if while updating the snapshots, some of the snapshots cannot find free disk blocks? That means those particular snapshots will no longer be able to recover to what it was when snapshots were taken.

+ Is the following answer to the question correct?
    + In a typical FreeBSD file system, there are 20 snapshot files allowed, which means, every time user tries to make a modification to the disk blocks that are "not yet copied", FS needs to check all those 20 snapshots to see if any of the snapshot file needs to do a copy (i.e. write to disk). Why?
        + In order to answer this question, we first need to understand what snapshots are used for. Snapshots stores various versions of what the disk partition was like at the time such snapshot was taken. Given a snapshot, we are able to recover the disk partition to be exactly what is described by the snapshot.
        + Hence, no matter what application tries to modify the disk block, such modification will be stored by every snapshot, so that for each snapshot, it stores this particular modification so that FS of this disk partition will be able to recover to any of the snapshots given this single modification.

+ Whenever a file is removed, the snapshot code inspects each of the blocks being freed and claims any that were in use at the time of the snapshot. Those blocks marked "not used" are returned to the free list.





What is the difference?
ln –s /usr/src/sys/sys/proc.h ppp.h
ln /usr/src/sys/sys/proc.h ppp.h



+ Hard link(di_nlink) && Symbolic link (shortcut).



What is the largest file an inode can handle?

Examples of the file systems: FAT32 (windows) - limitation of the file size is 4GB, NTFS, exFAT. So if you format your USB disk with FAT32, then the largest single file that you can store is 4GB regardless of how large your USB is. If instead, you format your USB disk with exFAT or NTFS, then you can break that file size limit.

hard link vs symbolic link?

How many disk blocks can a FS have?
264 or 232: Pointer (to blocks) size is 8/4 bytes.
How many levels of i-node indirection will be necessary to store a file of 2G (231) bytes? (I.e., 0, 1, 2 or 3)
12*210 + 28 * 210 + 28 *28 *2 10 + 28 * 28 *28 *2 10 >? 231 
What is the largest possible file size in i-node?
12*210 + 28 * 210 + 28 *28 *2 10 + 28 * 28 *28 *2 10
264 –1
232 * 210

What is the size of the i-node itself for a file of 10GB with only 512 MB downloaded?