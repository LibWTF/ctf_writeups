# Intro
For this challenge a "large file" exists in /home/big on a remote server. The challenge states that there is a present within the large file. Based on this information, we need to determine what important bytes are in this file hoping that it's the flag. Tools such as dd are also restricted on the remote host.

## Initial Research
After SSH'ing into the box we examine the size of the file:
```bash
big@ee6e73167aa0:~$ ls -lah my_huge_file 
-rw-r--r-- 1 root root 16T May 10 08:33 my_huge_file
```
We try to examine the contents of the file:
```bash
big@ee6e73167aa0:~$ cat my_huge_file 
😏^C
```
We notice that we can scp files to a tmp directory that we create for the team:
```bash
scp -P 9000 dd big@sharkyctf.xyz:/tmp/libwtf/
```
Now we can run dd locally to carve out a 4096 bytes of the file:
```bash
./dd if=/home/big/my_huge_file skip=1 bs=1 count=4096 >>1blk
```
Since some hex editors were restricted we decided to use od instead. The od command in Linux is used to convert the content of input in different formats with octal format as the default format. We can see the existence of a lot of zeros:
```bash
big@ee6e73167aa0:/tmp/libwtf$ od -b 1blk 
0000000 237 230 217 000 000 000 000 000 000 000 000 000 000 000 000 000
0000020 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000 000
*
0010000
```

## More research
After doing some googling we found this https://wiki.archlinux.org/index.php/sparse_file. A sparse file is a type of computer file that attempts to use file system space more efficiently when blocks allocated to the file are mostly empty. This is achieved by writing brief information (metadata) representing the empty blocks to disk instead of the actual "empty" space which makes up the block, using less disk space. The full block size is written to disk as the actual size only when the block contains "real" (non-empty) data.

Knowing this we can get the real size of the data within the file using linux tools:
```bash
big@ee6e73167aa0:~$ du -h my_huge_file 
32K	my_huge_file
```

We now know the data within the file is very small and we need to carve it out. But, how can we do this effectively? Luickly we have the following option params in the lseek function to speed this up (http://man7.org/linux/man-pages/man2/lseek.2.html).
```bash
SEEK_DATA - Adjust the file offset to the next location in the file greater than or equal to offset containing data. 
If offset points to data, then the file offset is set to offset.

SEEK_HOLE - Adjust the file offset to the next hole in the file greater than or equal to offset.  If offset points
into the middle of a hole, then the file offset is set to offset.  If there is no hole past offset, then the file
offset is adjusted to the end of the file (i.e., there is an implicit hole at the end of any file).
```

## The Solution
Luckily we found a C program online to find the offsets of non-zero data within the large file instead of writing my own. Shoutout to xkikeg https://gist.github.com/xkikeg/4645373.

After copying and compiling the C program to find data positions of sparse file with SEEK_DATA & SEEK_HOLE we can run it on the remote system. The following values are returned:
```bash
0x0 0x1000
0x47868c00000 0x47868c01000
0x77359400000 0x77359401000
0xa6e49c00000 0xa6e49c01000
0xe57a5680000 0xe57a5681000
0xffffe380000 0xffffe381000
0xfffff708000 0xfffff708042 - 17,592,176,640,000
```

Now that we have the offsets of where data exists we can use our local version of dd to carve out the data. The skip value is the start of each section and the count values is the start position of the data subtracted by the end position. We also converted the hex data to decimal. 

Example for the computing the offset of last data section within the file - 0xfffff708000 = 17,592,176,640,000:
```bash
big@ee6e73167aa0: ./dd if=/home/big/my_huge_file skip=17592176640000 bs=1 count=4096 >>outfin
```

When we do this for all blocks we get the flag:
```bash
big@ee6e73167aa0:/tmp/libwtf$ cat outfin 
😏shkCTF{sp4rs3_f1l3s_4r3_c001_6cf61f47f6273dfa225ee3366eacb8eb}
```
