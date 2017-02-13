---
layout: post
title: Testing applications with qemu user mode, automake, and buildroot
---

Previously I discussed [using qemu user mode to run cross-compiled binaries](http://yurovsky.github.io/2016/12/14/qemu-user-mode/), now let's put a few things
together and run unit tests automatically with [automake's `make check`](https://www.gnu.org/software/automake/manual/html_node/Tests.html). Then
we'll integrate everything into a build system (buildroot) and automatically
run cross-compiled tests as part of the build process. This has a number of
advantages including:

* no need to build for both x86 and ARM
* no need to deploy cross-compiled unit tests to a real target or even a `qemu-system-arm` machine, they'll just run on your machine
* easy integration and automation in your build system without requiring any additional steps or configurations

Many thanks to [Andrey Smirnov](https://github.com/ndreys) who explored this
further after discussing qemu user mode with me and figured all of this out
(I'm mostly just writing up notes on what he did).

# Running tests with automake

automake has a feature called `LOG_COMPILER` as part of its [Parallel Test Harness](https://www.gnu.org/software/automake/manual/html_node/Parallel-Test-Harness.html) infrastructure. If this variable is set, tests are executed
with the `LOG_COMPILER` as a wrapper. The Parallel Test Harness documentation
gives `perl` as an example wrapper but we can take advantage of this feature to
wrap our tests with a shell script calling out to qemu in order to run them
directly on our build machine.

We can capture and set this variable from the environment in `configure.ac`,
for instance:

    AC_SUBST([LOG_COMPILER], ${LOG_COMPILER})

It could be set like this when invoking `configure`:

    LOG_COMPILER=qemu-wrapper.sh ./configure

Of course we'd also include our cross-compilation options as well. We can then
run `make` and `make check` and we should see that the tests invoked by `make check` are now run using `qemu-wrapper.sh`.  That in turn needs to be some shell
script that runs its first command-line argument using `qemu-arm` with appropriate options (such as `-L`) set.

# Running tests with buildroot

Now let's put everything together into a buildroot recipe. I assume you're
familiar with how buildroot works (if not, please consult [the buildroot manual](https://buildroot.org/downloads/manual/manual.html) and familiarize yourself with how `external.mk` and recipes work).

## Add a test runner

We need something simple to point `LOG_COMPILER` to so we can add that as a
shell script somewhere in your `external/` tree. For example, `external/test-runner.sh` could look like:

    #!bin/sh
    
    exec ${HOST_DIR}/usr/bin/qemu-arm -L ${STAGING_DIR} $1

Keep in mind that `HOST_DIR` points to your build output's `host` directory,
containing the host's toolchain and, in this case, also the `host-qemu` executible. Meanwhile the `STAGING_DIR` gives us exactly what we need for qemu's `-L`
option.

We can then create a variable that points to this script. For instance, add:

    TEST_RUNNER := $(BR2_EXTERNAL)/test-runner.sh

to `external.mk`. Now package recipes have access to the `test-runner.sh`
script via `TEST_RUNNER`.

## Making sure qemu-arm is available

Enable qemu from the `Host Utilities` menu.  Specifically you want to turn on:

* host qemu
* Enable Linux user-land emulation

You don't need "Enable system emulation" unless you want to use qemu's machine
emulator mode.

Packages needing this qemu-based test infrastructure simply need to select
`BR2_PACKAGE_HOST_QEMU` as a dependency in their `Config.in`. For example,

    config BR2_PACKAGE_EXAMPLE_APP
        bool "example-app"
        select BR2_PACKAGE_HOST_QEMU
        help
          This is a our example app.

They also need to call out `host-qemu` as a dependency in their recipe, for
example for in `example-app.mk`, we could have:

    EXAMPLE_APP_DEPENDENCIES = host-qemu

Now when `example-app` is built, buildroot will build and deploy `qemu-arm`
(to `host`).

## Adding test hooks to a recipe

Given a typical buildroot recipe for our example app, `example-app.mk`, all
we need to do is set `LOG_COMPILER` for automake and tell buildroot to run
`make check` as a post-build step.  Setting `LOG_COMPILER` is easy by adding
to buildroot's `<package>_CONF_ENV` variable:

    EXAMPLE_APP_CONF_ENV += LOG_COMPILER=$(TEST_RUNNER)

Next, we add a post-build hook:

    define example-app-make-check
    $(MAKE) -C $(@D) check
    endef
    
    EXAMPLE_APP_POST_BUILD_HOOKS += example-app-make-check

Now we should see two things:

* buildroot will add our `LOG_COMPILER` variable to the environment when running `configure` in this package
* having built the package, buildroot will execute `make check` in the package build directory and that in turn will run any tests the package provides with `qemu-arm`

To try this out, build the package:

    make example-app

You should hopefully see the familiar `make check` output after the application
is built. Running `file` on binaries in the build directory should reveal that
they are in fact ARM binaries!

A failed unit test in the above example will break the build for that package.
You can adjust the implementation slightly if that's not what you want and you
could even introduce a configuration option in your `external/Config.in` menu
to make tests optional or control whether test failures break the build, as
usual there is a lot that can be customized with buildroot.

## Specifying which tests to run in a recipe

We may not want to run every test in the build system. Generally speaking, we
can run most unit tests but we will need to skip shell-based functional tests.
`automake` provides a way to do that with the `TESTS` environment variable.
For example, if we want to run only "test1" and "test2" we can specify them
like this:

    define example-app-make-check
    TESTS="test1 test2" $(MAKE) -C $(@D) -e check
    endef
    
    EXAMPLE_APP_POST_BUILD_HOOKS += example-app-make-check

The `-e` argument tells `make` to override variables from the environment and
we override `TESTS` (the variable calling out which tests to run) with our own
list. Now `make check` will run just those tests.

## qemu and kernel version

The recipe the builds qemu in buildroot, `package/qemu/qemu.mk`, performs a check comparing the host's kernel version to the target's and will
only succeed if the host version is greater than or equal to the target's. You
may find that buildroot refuses to build `qemu-arm` for you. The reasoning is
explained in `qemu.mk`: in user mode, qemu translates system calls, so it's assumed that the target does not make calls that don't exist on your host machine.

If this is a problem, you can defeat this check (`HOST_QEMU_COMPARE_VERSION`) at your own risk or upgrade your host machine if possible. Alternately, you can
use your own qemu rather than buildroot's (for example a distribution package or your own build). Generally speaking, it's unlikely that a unit test is going to make system calls that don't exist on your host machine but of course it depends on what you're testing and anything is possible.
