# Interrupt-Driven Socket I/O

The SIGIO signal notifies a process when a socket, or any file descriptor, has finished a data transfer. The steps in using SIGIO are as follows:

    Set up a SIGIO signal handler with the signal(3C) or sigvec(3UCB) calls.

    Use fcntl(2) to set the process ID or process group ID to route the signal to its own process ID or process group ID. The default process group of a socket is group 0.

    Convert the socket to asynchronous, as shown in Asynchronous Socket I/O.

The following sample code enables receipt of information on pending requests as the requests occur for a socket by a given process. With the addition of a handler for `SIGURG`, this code can also be used to prepare for receipt of `SIGURG` signals.

Example 8–13 Asynchronous Notification of I/O Requests
```
#include <fcntl.h>
#include <sys/file.h>
 ...
signal(SIGIO, io_handler);
/* Set the process receiving SIGIO/SIGURG signals to us. */
if (fcntl(s, F_SETOWN, getpid()) < 0) {
    perror("fcntl F_SETOWN");
    exit(1);
}
```
