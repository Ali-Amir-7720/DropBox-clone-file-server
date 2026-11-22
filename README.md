**Dropbox Clone: Multi-Threaded File Storage Server**
A small Dropbox-like file storage server written in C for the Operating Systems course (Fall 2026). Users sign up, log in, and then upload, download, delete and list their files over TCP, with a fixed storage quota per user. Many clients can be connected at the same time, and one user may be logged in from several clients at once.

The project is about concurrency structure rather than networking: every request passes through two thread-safe queues and two thread pools before it touches the disk, and the result is delivered back to the exact client session that asked for it.

Architecture
 Clients            Main          Client        Client          Task         Worker         Shared state
 (processes)       thread         Queue       Threadpool        Queue      Threadpool
                                                                                          storage/<user>/<file>
  client A  --(1)--> accept() --(2)--> [fd] --(3)--> CT 1 --(4)--> [task] --(5)--> W 1 --(6)--> metadata
  client B                                           CT 2                          W 2          lock table
  client C                                           CT 3                          W 3
     ^                                                 ^                             |
     |                                                 +----(7) result + signal -----+
     +------------------(8) reply (client threads only touch sockets)
Main thread only calls accept() and pushes each new socket into the Client Queue.
Client Queue: bounded, thread-safe queue (mutex + condition variables) of socket descriptors.
Client Threadpool: each thread owns one session at a time. It handles signup/login, reads commands, builds a Task, pushes it to the Task Queue, waits for the result and writes the reply. Client threads never do file operations.
Task Queue: second instance of the same queue type, holding tasks.
Worker Threadpool: workers pop tasks, take the needed locks, check the quota, do the file I/O, update metadata and hand the result back to the waiting client thread. Workers never touch sockets.
Both queues are bounded, so a slow disk causes backpressure instead of unbounded memory growth.

Design decisions
Open problem	Chosen design
Worker to client result delivery	e.g. per-task mutex + condition variable + done flag
Same-user conflicts (locking)	e.g. per-user rwlock; lock order: users table, then user
Waiting without busy-waiting	e.g. pthread_cond_wait in a while loop on done
Metadata storage	in memory, rebuilt from storage/ at startup / small file
Building
Requires gcc, make, POSIX threads. For the test targets: valgrind and a gcc with ThreadSanitizer.

make            # builds ./server and ./client
make tsan       # same sources built with -fsanitize=thread
make clean      # removes objects, binaries and storage/
Compiler flags:

CC     = gcc
CFLAGS = -std=c11 -D_DEFAULT_SOURCE -Wall -Wextra -Werror -g -O0 -pthread
Running
Start the server:

./server <port> <client_threads> <worker_threads>
# example
./server 8080 16 4
Connect with the client (reads commands from standard input, so it can be scripted):

./client <host> <port>
# example
./client 127.0.0.1 8080
Example session:

SIGNUP alice secret1
OK signed up
LOGIN alice secret1
OK logged in
UPLOAD notes.txt
OK uploaded
LIST
OK 1
notes.txt 2048
DOWNLOAD notes.txt
OK 2048
DELETE notes.txt
OK deleted
QUIT
OK bye
UPLOAD <file> reads the local file and sends it with its size; DOWNLOAD <file> saves the received bytes to a local file of that name.

Stop the server with Ctrl+C. It shuts down gracefully: stops accepting, drains or discards queued work, joins all threads, closes every socket and frees all memory.

Protocol
Every command is one text line ending in \n. File contents are sent as a size followed by exactly that many bytes.

Client sends	Server replies
SIGNUP <user> <password>	OK signed up or ERR EXISTS
LOGIN <user> <password>	OK logged in or ERR AUTH
UPLOAD <filename> <size> + <size> bytes	OK uploaded, ERR QUOTA, ERR EXISTS or ERR SIZE
DOWNLOAD <filename>	OK <size> followed by exactly <size> bytes, or ERR NOTFOUND
DELETE <filename>	OK deleted or ERR NOTFOUND
LIST	OK <n> followed by n lines of <filename> <size>
QUIT	OK bye, then the connection is closed
Anything except SIGNUP/LOGIN/QUIT before login	ERR NOAUTH
Unknown command or wrong number of arguments	ERR BADCMD
Limits and validation
Quota: 1 MB per user; maximum file size: 256 KB (both #defined in include/config.h).
File names: flat, at most 63 characters, only letters, digits, ., _ and -. Names containing / or .. are rejected.
User names: 3 to 31 letters or digits.
Every size, length and count received from the network is validated.
Passwords are never printed to logs.
Each user's files are stored in storage/<user>/.
Testing
make test       # runs the test scripts in tests/
make valgrind   # runs the server under Valgrind for the test scenario
make demo       # runs the whole suite and prints ThreadSanitizer and Valgrind summaries
The suite (tests/run_all.sh) covers:

Single client: signup, login, upload, list, download (checked with cmp), delete.
Many clients: 10+ concurrent clients, each with its own user, doing mixed operations.
Same user, many sessions: 5+ sessions of one user hammering the same file names; at the end LIST, the files on disk and the quota used must agree.
Quota race: concurrent uploads where only some fit; the user never goes over quota and exactly the uploads that fit succeed.
Bad input: unknown commands, names like ../x, oversized files, commands before login, and a client that disconnects mid-upload.
Shutdown: SIGINT while clients are connected.
Project structure
.
├── Makefile
├── README.md
├── include/        headers (config.h holds the limits)
├── src/            server sources
├── client/         client program
├── tests/          test scripts and run_all.sh
└── report.pdf      2-page design report
Synchronization primitives
Only POSIX/C11 primitives are used: pthread mutexes, condition variables, read-write locks, unnamed semaphores and C11 atomics. Which one protects what, and the lock order, is documented in a comment above the lock definitions in the source and in report.pdf.

Course
Operating Systems, Computer Science, Fall 2026 – semester project.
