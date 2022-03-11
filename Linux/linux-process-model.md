# The Linux Process and Session Model

Author: Mike Sample

Linux follows the Unix process model from the 1970's that was augmented
with the concept sessions in the 1980's (as near as I can tell from
when the `setsid()` system call was introduced by early [POSIX
documents](https://en.wikipedia.org/wiki/POSIX)).

## Process Model Capture vs System Call Logs

Capturing changes to the session model in terms of new processes, new
sessions, exiting processes etc. is simpler and clearer than capturing
the [system calls](https://en.wikipedia.org/wiki/System_call) used to
enact those changes.  Linux has approximately 400 system calls and
does not refactor them once they are released. This approach retains a
stable application binary interface (ABI) which means programs
compiled to run on Linux years ago should continue to run on Linux
today without rebuilding them from source code. New system calls are
added to improve capabilities or security instead of refactoring
existing system calls (avoids breaking the ABI).  The upshot is that
mapping a time ordered list of system calls and their parameters to
the logical actions they perform takes a significant amount of
expertise.  Additionally, newer system calls such as those of io_uring
make it possible to read and write files and sockets with no
additional system calls by using memory mapped between kernel and user
space. By contrast, the process model is stable (hasn't changed much
since the 1970's) yet still comprehensively covers the actions taken
on a system when one includes file access, networking and other
logical operations.

## Process Formation: init is the first process after boot

When the Linux kernel has started it creates a special process called
[“the init process”](https://en.wikipedia.org/wiki/Init).  A process
embodies the execution of one or more programs.  The init process
always has the process id (PID) of 1 and is executed with a user id of
0 (root).  Most modern Linux distributions use systemd as their init
process's executable program.  The job of init is to start the
configured services such as databases, web servers, and remote access
services such as sshd.  These services are typically encapsulated
within their own sessions which simplifies starting and stopping
services by grouping all processes of each service under a single
session id (SID).

Remote access, such as via the SSH protocol to an sshd service, will
create a new Linux session for the accessing user.  This session will
initially execute the program the remote user requested, often an
interactive shell, and the associated process(es) will all have the
same SID.

## The Mechanics of Creating a Process

Every process, except the init process, has a single parent process.
Each process has a PPID, the process id of its parent process
(0/no-parent in the case of init).  Reparenting can occur if a parent
process exits in a way that does not also terminate the child
process(es). Reparenting usually picks init as the new parent and init
has special code to clean up after these adopted children when they
exit. Without this adoption and clean up code, orphaned child
processes would become "zombie" processes - no kidding. They hang
around until their parent reaps them so the parent can examine their exit
code, an indicator of whether the child program completed its tasks
successfully. The advent of "containers", pid namespaces in
particular, necessitated the ability to designate processes other than
init as "sub-reapers" (processes willing to adopt orphaned
processes). Typically sub-reapers are the first process in a
container; this is done because the processes in the container cannot
"see" processes in the ancestor pid namespaces (i.e. their PPID value
would not make sense if the parent was in an ancestor pid namespace).

To create a child process, the parent clones itself via the [`fork()`
or `clone()` system
call](https://en.wikipedia.org/wiki/Fork_(system_call)).  After the
fork/clone, execution immediately continues in *both* the parent *and*
the child (ignoring `vfork()` and `'clone()`'s `CLONE_VFORK` option),
but along different code paths by virtue of the return code value from
`fork()`/`clone()`. You read that correctly – one `fork()`/`clone()`
system call provides a return code in two different processes! The
parent receives the PID of the child as its return code, and the child
receives 0 so the shared code of the parent and child can branch based on
that value. There are some cloning nuances with multi-threaded parents
and copy-on-write memory for efficiency that do need to be elaborated
on here.  The child process inherits the memory state of the parent
and open files, network sockets and the controlling terminal, if any.

Typically the parent process will capture the PID of the child to
monitor its lifecycle (see reaping above).  The child process's
behavior depends on the program that cloned itself (it provides
execution path to follow based on the return code from `fork()`).

A web server such as nginx might clone itself, creating a child
process to handle http connections.  In cases like this, the child
process does not execute a new program but simply runs a different
code path in the same program, to handle http connections in this
case. Recall that the return value from clone or fork tells the child
that it is the child so it can choose this code path.

Interactive shell processes (e.g. one of bash, sh, fish, zsh,
etc. with a controlling terminal), possibly from an ssh session, clone
themselves whenever a command is entered. The child process, still
running a code path from the parent/shell, does a bunch of work
setting up file descriptors for IO redirection, setting the process
group, and more before the code path in the child calls the
[`execve()`](https://man7.org/linux/man-pages/man2/execve.2.html)
system call or similar to run a different program inside that process.
If you type `ls` into your shell, it forks your shell, the setup
described above is done by the shell/child and then the `ls` program
(usually from the `/usr/bin/ls` file) is executed to replace the
contents of that process with the machine code for `ls`. This
[article](https://www.gnu.org/software/libc/manual/html_node/Implementing-a-Shell.html)
about implementing shell job control provides great insight into the
innerworkings of shells and process groups.

It is important to note that a process can call `execve()` more than
once, and therefore workload capture data models must handle this as
well.  This means that a process can become many different programs
before it exits – not just its parent process program optionally
followed by one program.  See the shell [`exec` builtin
command](https://www.man7.org/linux/man-pages/man1/bash.1.html#SHELL_BUILTIN_COMMANDS)
for a way to do this in a shell (i.e. replace the shell program with
another in the same process).  Another aspect of executing a program
in a process is that some open file descriptors (those marked as
close-on-exec) may be closed prior to the exec of the new program,
while others may remain available to the new program.  Recall that a
single `fork()`/`clone()` call provides a return code in two
processes, the parent and the child.  The `execve()` system call is
strange as well in that a successful `execve()` has no return code for
success because it results in a new program execution so there's
nowhere to return to execept when `execve()` fails.

## Creating New Sessions

Linux currently creates new sessions with a single system call,
`setsid()`, which is called by the process that becomes the new
session leader.  This system call is often part of the cloned child’s
code path run before exec’ing another program in that process
(i.e. it’s planned by, and included in, the parent process’s code).
All processes within a session share the same SID, which is the same
as the PID of the process that called `setsid()`, also known as the
session leader. In other words, a session leader is any process with a
PID that matches its SID.  The exit of the session leader process will
trigger termination of its immediate children process groups.

## Creating New Process Groups

Linux uses process groups to identify a group of processes working
together within a session. They will all have the same SID and process
group id (PGID).  The PGID is the PID of the process group
leader. There is no special status for the process group leader; it
may exit with no effect on other members of the process group and they
retain the same PGID, even though the process with that PID no longer
exists.  Note that even with pid-wrap (re-use of a recently used pid
on busy systems), the Linux kernel ensures the pid of an exited
process group leader is not reused until all members of that process
group have exited (i.e. there is no way their PGID could accidentally
refer to a new process).


Process groups are valuable for shell pipeline commands like:

```shell
cat foo.txt | grep bar | wc -l
```

which creates three processes for three different programs (`cat`,
`grep` and `wc`) and connects them with pipes.  Shells will create a
new process group even for single program commands like `ls`.  The
purpose of process groups is to permit targeting of signals to a set
of processes and to identify a set of processes, the foreground
process group, that are permitted full read and write access to their
session’s controlling terminal, if any. In other words control-C in
your shell will send an interrupt signal to all processes in the
foreground process group (the negative PGID value as the signal’s pid
target discriminates between the group versus the process group leader
process itself).  The controlling terminal association ensures that
processes reading input from the terminal don’t compete with each
other and cause issues (terminal output may be permitted from
non-foreground process groups).

## Users and Groups

As mentioned above, the init process has the user id 0 (`root`).
Every process has an associated user and group and these may be used
to restrict access to system calls and files. Users and groups have
numeric ids and *may* have an associated name like `root` or `ms`.
The `root` user is the superuser which may do anything, and should
only be used when absolutely required for security reasons.  The Linux
kernel only cares about ids. Names are optional and provided for human
convenience by the files
[`/etc/passwd`](https://man7.org/linux/man-pages/man5/passwd.5.html)
and
[`/etc/group`](https://man7.org/linux/man-pages/man5/group.5.html). The
[Name Service Switch
(NSS)](https://man7.org/linux/man-pages/man5/nsswitch.conf.5.html)
allows these files to be extended with users and groups from LDAP and
other directories (use [`getent
passwd`](https://man7.org/linux/man-pages/man1/getent.1.html) if you
want to see the combination of /etc/passwd and NSS-provided users).

Each process may have several users and groups associated with it
(real, effective, saved, and supplemental groups). See [`man 7
credentials`](https://man7.org/linux/man-pages/man7/credentials.7.html)
for more information.

The increased use of containers whose root file systems are defined by
container images has increased the likelihood of /etc/passwd and
/etc/group being absent or missing some names of user and group ids
that may be in use.  Since the Linux kernel does not care about these
names, only the ids, this is fine.

## Learning More

Linux man pages are an excellent source of information. The man pages
below have details of the Linux process model described above:

* [man 2 fork](https://man7.org/linux/man-pages/man2/fork.2.html)
* [man 2 clone](https://man7.org/linux/man-pages/man2/clone.2.html)
* [man 2 setsid](https://man7.org/linux/man-pages/man2/setsid.2.html)
* [man 2 execve](https://man7.org/linux/man-pages/man2/execve.2.html)
* [man 2 exit](https://man7.org/linux/man-pages/man2/_exit.2.html)
* [man 7 credentials](https://man7.org/linux/man-pages/man7/credentials.7.html)
* [man 7 namespaces](https://man7.org/linux/man-pages/man7/namespaces.7.html)
* [process groups - implementing shell job control](https://www.gnu.org/software/libc/manual/html_node/Implementing-a-Shell.html)
* [shell builtin commands](https://www.man7.org/linux/man-pages/man1/bash.1.html#SHELL_BUILTIN_COMMANDS)
