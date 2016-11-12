---
layout: post
title: Opening files (and Linux drivers)
---

We should be careful (if not downright paranoid) when opening files. Some of
the recommended checks to perform and best practices include:

 * Using [realpath(3)](http://man7.org/linux/man-pages/man3/realpath.3.html) to product a canonicalized absolute path for us when the path comes from outside.
 * Setting O\_CLOEXEC to make sure that the file descriptor is not leaked to any child processes we spawn
 * Sanity-checking the file with [fstat(2)](http://man7.org/linux/man-pages/man2/stat.2.html) to make sure it looks like the kind of file we are expecting

When appropriate, we can also use the fstat(2) output to check for:

 * Expected file ownership (at least with the `st_uid` and `st_gid`), for example if we expect a specific user and/or group to own the file, regardless of our ability to access it.
 * An expected physical location (`st_dev` field), for example if we expect it to reside on a specific disk or device regardless of what the path says.
 * Expected permissions (`st_mode` field), for example we may want to refuse to open an executable file or a file with too lax of permissions no matter what

# The path

A program taking a path from the outside world, for example through a command
line argument, should sanitize the path (per CERT [FIOO2-C](https://securecoding.cert.org/confluence/display/c/FIO02-C.+Canonicalize+path+names+originating+from+tainted+sources)).  Typically this looks like:

    int main(int argc, char **argv)
    {
        if (argc < 2) {
            printf("Usage: %s <path>\n", argv[0]);
            exit(EXIT_SUCCESS);
        }

        char *path = realpath(argv[1], NULL);
        if (!path) {
            fprintf(stderr, "Invalid argument: %m");

The resolved path will need to be freed.

# Example: Talking to a SPI (spidev) device

We can now try to open the file given the resolved path, however we should
sanity-check that the file looks like something that we expect to open.  For
example, a SPI device interfaced via the Linux spidev driver should at least:

* Be a character device (and not, say, a regular file or socket)
* Have the major 153

We specifically want to use fstat(2) rather than stat(2) because we want to
be sure that we are checking the file we opened rather than checking a path
twice (and allowing whatever the path resolves to to change out from under
us).

First, we open the file:

    int fd = open(path, O_CLOEXEC);
    if (fd == -1) {
        fprintf(stderr, "Unable to open \"%s\"\n", path);
        goto out_fd;
    }

Then we perform some basic sanity checks. The `st_mode` field tells us what
kind of file we have and there are convenient macros to simplify our checks:
`ST_IS_CHR` checks that this is a character device, which is what we expect.
In order cases we may want a regular file (`S_ISREG` matches that).

For character devices and other special files we also have the `st_rdev` field
on which we can use macros `major` and `minor` to check for specific IDs.

    struct stat st;
    if (fstat(fd, &st) || !S_IS_CHR(st.st_mode) || major(st.st_rdev) != 153) {
        fprintf(stderr, "\"%s\" does not look like a SPI device\n", path);
        goto out_st;
    }

We have now opened the file and reasonably believe that it is a SPI device.
Our cleanup would be in reverse order:

    out_st:
        close(fd);
    out_fd:
        free(path);

or the GCC [cleanup attribute](https://gcc.gnu.org/onlinedocs/gcc/Common-Variable-Attributes.html) can be used to clean up resources.
