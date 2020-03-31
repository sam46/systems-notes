# OS NOTES  
  
### Garbage Collection
##### reference counting
##### tracing
- mark-and-sweep
- mark-and-copy

### ELF
- portable format!
- file header: info like 64 vs 32bit, target OS (Linux, FreeBSD, Solaris, System V, ...)  
- sections: actual data or metadata: .text, .data, .rdata  
- section header: refrences the sections. used by linker (and dynamic linker). describes sections  
- program header: refrences the sections. used by OS/loader. describes memory segments  

a memory segment contains 1 or more sections

#### Program header: 
- describes runtime execution (memory segments)  
- tells the system how to create a process image  
- specifies memory segments (size in file and in memory, offset in file and in virtual memory)  
- etc...  

#### Section header:
- used by linker/dynamic-linker/relocation  
- refrences 0 or more sections  


### Loader
uses mmap to lazily load binary from disk and uses the page cache (see below)  
- binaries/libraries are shared between process automagically  
- unused parts of binaries/libraries are automatically flushed out of memory  

### Virtual Memory

#### MMAP
- lazy by default  
- can be set to eager by passing POPULATE flag, or   
  using mlock (pages are guaranteed to always be resident in RAM)  
- a usecase: reading files... faster because it avoids many `read` syscalls    

##### shared vs private:
- shared: Share this mapping. Updates to the mapping are visible to  
              other processes mapping the same region, and (in the case of  
              file-backed mappings) are carried through to the underlying  
              file.  
            usecases: memorymapped I/O and IPC  
- private: Create a private copy-on-write mapping. Updates to the  
              mapping are not visible to other processes mapping the same  
              file, and are not carried through to the underlying file  

##### anonymous vs file-backed:
anonymous is:  
- not backed by any file  
- its contents are initialized to zero  
malloc uses anon mmap!  

### Page cache/buffer cache
- see filesystem cache below!
- swap has nothing to do with the page cache: program data saved to disk. Completely different!  
- non-dirty page cache pages are the first to go if system is tight on memory. They're just discarded since they're already on disk.  
- most file I/O is driven by virtual memory's page cache:
  try running `free -h` before and after running `python -c "print 'a'*(4000*2**20)" > ~/f.bin` 
- flush page cache: `echo 1 > /proc/sys/vm/drop_caches` 

### Swap
store processes data on disk    
`free -h`

**should be able to explain most of `/proc/meminfo`**  
[https://github.com/torvalds/linux/blob/master/Documentation/filesystems/proc.txt](https://github.com/torvalds/linux/blob/master/Documentation/filesystems/proc.txt)

shared memory (shmem):
- shared memory + `tmpfs` (`df` to see all `tmpfs` mounts)
- `shared` column in `free`
- `tmpfs` is basically ram disk

## Networking

### TCP/IP model vs OSI model
There's some confilict, but generally, OSI is more general.   
Mapping from TCP/IP to OSI:  
- Application -->  [application, presentation layer, and most of session]
- Transport --> [transport, the graceful close function of the session layer]
- Internet --> subset of Network
- Link --> [data link layer, physical layer, some protocols of the network layer]

### OSI
note: numbered layers are NOT USED in TCP/IP model  
#### L4 Transport Layer
Responsible for delivering data to the appropriate application process on the host computers.
Provides quality of life features:  
- reliablility  
- connection-oriented communication (in case of TCP)    
- flow control: rate of data transmission between two nodes    
- congestion avoidance: prevents buffer-overrun    
protocols: TCP, UDP (higher throughput lower latency, but packet loss ok), ...  

##### UDP
connection-less, can boradcast and multicast too:  
- boradcast: sent packets can be addressed to be receivable by all devices on the subnet  
- multicast: packet can be sent to very large numbers of subscribers  

#### L3 Network Layer
Host addressing (e.g. ip) and cross-network message forwarding (e.g. through routers/gateways)  
protocols: IPv4/IPv6, ICMP, WireGuard, IPsec  


## File systems (FS)

File:
- regular file
- special file
  - block
  - character
  - named pipe (aka. FIFO file): in-memory like regular pipes, but have a file discriptor, so can be shared by processes
  - socket
    - Unix Domain Socket
    - Network Socket (TCP/stream, UDP/datagram, RAW)

Some steps required to write data to end of some file:  
- find empty disk blocks and mark them as in use  
- associate these blocks with the file  
- adjust file size  
- actually copy the data to the blocks  
  
at least 3 datastructures are needed:  
- for tracking free disk blocks  
- for tracking which data blocks belong to a file (Inode!)  
- the data blocks themselves  

### Inode (aka index inode)
  
An inode:  
- each inode is named and located by a number: `ls -i`
- stores location of file's data blocks on disk  
- stores file metadata: permissions, various timestamps  
  
ext4:   
- amount of inodes is determined at format time!  
- inode takes space on disk (256 bytes == 1/16th of a block)  
- tradoff: more inodes == more files, but less space for data blocks   
- default: 1 inode for every 16 KB of data blocks (configurable at format time)  
  
How to retrieve (a specific) disk block given an inode and offset?  
- multilevel indexing:   
inode points to data blocks and an "indirect block", indirect blocks points to more data blocks and an indirect block, so on. This works well (time & space) for large and small files.  
  
### Directory
- contains (filename --> inode number) mappings

steps for open("/etc/f.txt"):
- "/" probably has hardcoded inode num (2)
- fetch inode #2, lookup inode num associated with "etc" in the contents: x
- fetch inode #x, lookup inode num associated with "f.txt" in the contents: y
- now we know the inode num for target file!

##### hard vs soft (symoblic) link
hard link:
- filename to inode number (just like in directory)
- doesn't work cross-filesystems: because the inode num we map to is on the same fs.
- if original file is deleted or moved, hardlinks are unaffected and the file can still be accessed!
  
soft link:
- name --> name mapping
- works cross filesystems

#### Caching  
```
  applications  
      |  
     vfs  
    /   \  
   fs1  fs2   
    \   /  
     disk  
```
where to stick the cache?  
**above vfs:**  
- sees file contents only  
- doesn't see fs metadata (e.g. inodes)  
- `cache` (aka pagecache) in `/proc/meminfo`, `free`, and `vmstat`   

**below actual filesystem:**  
- sees disk blocks, i.e. file contents and metadata, but to avoid storing file contents twice (in `buffer` and in `cache`),  linux only stores file contents in `cache`, while the `buffer` points to `cache`  
- `buffers` in `/proc/meminfo`, `free`, and `vmstat`  
   
Caching policy:  
- write-through:  
writes to cache immediately make it to disk: slow writes, fast reads, safe  
- write-back:    
writes to cache make it to disk asynchronously: fast writes and reads, risky  
`sync`/`fsync` flush cache to disk  
  
Recovering from incosistencies:  
(writing a single disk block is assumed to be atomic!)  
- `fsck` checks entire file system: slow!  
- **journaling:**  
  - record changes to be made in a journal (a circular log)  
  - check them off as writes make it to disk (aka checkpointing)  
  - Everything before the last checkpoint is assumed to have safely
    made it to disk  
  - Anything after the last ckeckpoint may/may not have made it. We only need to check entries
    since last checkpoint  
  - much faster!  
  - ext4 is a journaling fs  
     
  
  
# Misc.

#### poll/select:
Need to read from (say) 3 file descriptors, each may block of it's not currently readable/writable (aka. ready)
naively:
```
read(fd1)
read(fd2)
read(fd3)
```
can't block on more than one fd at once (assuming 1 thread).
e.g. Might block on fd1 while fd2 is actually ready and a `read` wouldn't block.
  
- Solution, nonblocking IO:   
`read(fd)` returns error without blocking if fd currently not readable. Application needs to continuously poll, wasting cpu.
- Solution, multiplexed IO (`select`/`poll`):  
blocks on all fd's **at once**, then issue nonblocking read on the ready fd's:
```
select(fd1,fd2,...,timeout)
```
sleeps until an fd is ready, or a time out, then issue a `read` on the ready fd, which won't block.  
in other words: blocking (if we choose to) `select` on multiple fd's **at once**, followed by nonblocking read.  
note: 
  - depending on the timeout, `select` can block forever, return immediately, or until the timeout  
  - `select`/`poll` is O(n) in number of fds, `epoll` is more efficient!

#### level-triggred vs edge-triggred event and epoll
  
  
### Zombies
- parent process terminates before child ---> child's parent becomes `init`  
- When a process terminates, it is not immediately removed from the system but parts of the it
  are kept resident in memory to allow the parent to inquire about its status upon terminating (aka `wait`)  
- once parent `wait()`ed on the child process, it's fully destroyed  
- zombie: process that terminated but hasn't yet been `wait`ed upon  
- `init` routinely `wait`s on all of its children  