>Using this guide: https://beej.us/guide/bgnet/ I followed along and just tried to parse the knowledge for myself

## What is a Socket?
---
Essentially: a way to speak to other programs using standard Unix file descriptors. A [UNIX file descriptor](https://stackoverflow.com/questions/5256599/what-are-file-descriptors-explained-in-simple-terms) is what the OS uses to represent/differentiate files that are running. They get assigned unique entries. Every process on UNIX is essentially a file, since they all have file descriptors. Whenever a unix program does I/O, they do it by reading or writing to a file descriptor.

In more depth, each process is assigned a file descriptor (fd), they are integer values that when referenced open I/O resources. This is important because it creates a unified interface for all types of I/O (files, devices, sockets, pipes). Every process has its own independent file descriptor table. 
#### UNIX File Descriptors Rant
---
>This is new information, that is crucial to learning and understanding this
##### Standard File Descriptors
---
- fd 0: Standard input (stdin) - where processes read input from
- fd 1: Standard output (stdout) - where processes write normal output
- fd 2: Standard error (stderr) - where processes write error messages
- These three are automatically set up for every process at creation
##### How they are managed
---
- **Per-process file descriptor table**: Maps integers to system-wide file table entries
- **System-wide file table**: Contains all open file information across all processes
- **System-wide inode table**: Points to actual resources
##### Why this matters
---
- Creates composability through pipes and redirection
	- Since we know that fd 0 is stdin and fd 1 is stdout it makes it easy to redirect I/O operations
- Consistent resource management and access control
- Interprocess communication
##### How redirection operators work with file descriptors
---
Redirection operators in Unix shells directly manipulate file descriptors.
###### Basic Redirection 
###### Output Redirection (>)
---
```bash
ls > files.txt
```
This redirects file descriptor 1 (stdout) to a file named "files.txt".
The shell:
- Opens "files.txt" for writing, getting a new file descriptor (e.g., fd3)
- Uses ``dup2()`` to copy fd3 to fd1, closing the original fd1
	- ``dup2()`` makes a secondary pointer to the same resource, essentially
	- fd3 is just the most next available fd, so it just fills that space
- Closes fd3
- Executes ls, which write to its fd1(which is pointing to files.txt)
###### Input redirection (<)
---
```shell
sort < unsorted.txt
```
This redirects file descriptor 0 (stdin) to read from "unsorted.txt".
The shell:
- Opens "unsorted.txt" for reading, getting a new file descriptor
- Duplicates this to fd 0, replacing the original stdin
- Executes ``sort``, which reads from its fd 0 (now pointing to unsorted.txt)
	- ``sort`` is just a command that sorts through lines

The biggest takeaway from all of this is since they communicate through these File Descriptors, it is super abstracted so all these commands care about is getting input into its stdin (fd 0). The shell just redirects the processes stdin/stdout to just point to different places.

#### Two Types of Internet Sockets (not really)
---
>He recommends to look up "Raw Sockets", on top of the "Two Types of Internet Sockets".

Two Types of Sockets:
- Stream Sockets (S0CK_STREAM)
- Datagram Sockets (S0CK_DGRAM)
	- Datagram sockets are sometimes called, "connectionless sockets". 

Stream Sockets are reliable two-way connected communication streams. If you output two items into the socket in the order "1, 2", they will arrive in the order "1, 2" at the opposite end. They are also error-free.

``telnet`` and ``ssh`` both use stream sockets. Web browsers use HTTP, which uses stream sockets to get pages. If you telnet to a website on port 80, and type ``GET/HTTP/1.0`` and hit return twice, it spits the HTML back.

Stream Sockets achieve high level of data transmission quality due to its protocol, The Transmission Control Protocol (TCP).

Now for Datagram Sockets, they are unreliable, and can arrive out of order. Both 
datagram sockets and stream sockets use IP for routing, but datagram socket use User Datagram Protocol, or UDP.

They are "connectionless" because you don't have to maintain a connection, as you do with stream sockets. You just build a packet, slap an IP header on it with a destination and send it out. You just pray that it gets there. Generally used when TCP stack is unavailable or when a few dropped packets don't mean anything. Some examples are Trivial File Transfer Protocol, DHCP client, multiplayer games, streaming audio, video conferencing, etc.

Things like Trivial File Transfer Protocol have their own system built on top of UDP, which lets it work. Essentially it awaits for ACK packet to see if it got it, if it doesn't get that it resends until then.

For things like games or streaming, you simply ignore the packets and try to compensate for them.

You would use UDP for SPEED, its fast to just send and forget rather than to ensure it is being safely sent. So, you need TCP if it must be reliable.

#### Low Level Nonsense and Network Theory
---
