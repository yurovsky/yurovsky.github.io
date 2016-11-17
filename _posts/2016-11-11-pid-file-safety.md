---
layout: post
title: PID file safety
---

There are some instances where we need to use the old-style PID files in order
to signal a daemon process (typically sending it a `SIGHUP` to ask it to
re-read its configuration files or a `SIGTERM` to ask the daemon to exit). We
can get our of the business of managing daemons with things like systemd but in
the cases when we still have to deal with PID files, we should be careful.

Typically we have opened a PID file for a hypothetical daemon, `food`:

    FILE *f = fopen("/var/run/food.pid", "r");
    if (f) {
        /* Daemon may be running. */

We then need to read in the PID (we expect the file to contain just a number)
so we can send that daemon a signal.  This can be as simple as:

    pid_t pid;
    if (fscanf(f, "%" SCNiMAX, &pid) == 1) {
        /* We probably read a PID */

It's very important that we check the result of fscanf(3) as we only want to
proceed if we read something that looks like a number. [kill(2)](http://man7.org/linux/man-pages/man2/kill.2.html) is very
dangerous depending on what we do with it and what our privileges are on the
system.  For instance, the `pid` argument to kill implies:

* Send the signal to every process that we have permission to if `pid` is -1, certainly this would be bad.  If `pid` is less than -1, the signal will be sent to a particular process group.
* Send the signal to every process in the process group of the calling process if `pid` is 0, this is also bad.
* If `pid` is positive, send the signal to that process.  However if `pid` is 1, we are going to send that process to the init system. If `pid` happens to be our own PID, we send the signal to ourselves.

The data type for a PID is `pid_t` which in the GNU standard C library is an
`int`. We should probably check that the PID at least looks like something
sane:

    pid_t pid;
    if (fscanf(f, "%" SCNiMAX, &pid) == 1 && pid > 1 && pid != getpid()) {
        /* We probably read a PID and it seems sane */

Depending on the application, we may also use [getppid(2)](http://man7.org/linux/man-pages/man2/getpid.2.html) to make sure the PID is not the one for our own parent process.

We can then try to check if someone is running with that PID.  Traditionally
that is done by sending a 0 signal (a 0 signal means that no signal is actually
sent but error checking is still performed).

    fclose(f);
    
    if (kill(pid, 0) == -1) {
        if (errno == ESRCH) {
            /* Nothing with that PID seems to exist
               (or the process is a zombie) */
        } else if (errno == EPERM) {
            /* We don't have permissions to send a signal
               to this process */
        }
    } else {
        /* We can send signals to this PID, proceed,
           for example: */
        if (kill(pid, SIGHUP)) {
            fprintf(stderr, "Couldn't signal %d: %m", pid);
        }
    }

Finally, if we need to implement the daemon side (double-forking, closing file
descriptors, optionally creating PID files, and so on) please consider linking
against [libdaemon](http://0pointer.de/lennart/projects/libdaemon/) rather than
writing the code yourself (see the [testd.c example](http://0pointer.de/lennart/projects/libdaemon/reference/html/index.html) for usage).
