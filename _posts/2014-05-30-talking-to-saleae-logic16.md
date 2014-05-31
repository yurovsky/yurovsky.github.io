---
layout: post
title: Talking to the Saleae Logic16
---

I'm tracking down a few issues at work that require automating a logic
analyzer, in this case the task is as simple as monitoring a SPI bus and
looking for particular patterns.  The
[Saleae Logic16](https://www.saleae.com/logic16) is a decent low-cost USB
logic analyzer that can sample a four channels at 50MHz and provides a C++ API
that enables you to write a custom application.

I went ahead and downloaded their API (version 1.1.14 in this case),

    wget http://downloads.saleae.com/SDK/SaleaeDeviceSdk-1.1.14.zip
    unzip SaleaeDeviceSdk-1.1.14.zip
    cd SaleaeDeviceSdk-1.1.14

This provides a 32-bit and 64-bit library in ./lib as well as one header file
in ./include.  Saleae provides an example program in ./source but no Makefile
(I don't think they know how to write one, instead they provide a Python script
that tries to call g++ directly, quite a mess).

## "Installing" the SDK

Let's write an application to talk to this analyzer.  First copy the library
and header file and run "ldconfig" to let the system know about the former:

    sudo cp lib/*.so /usr/local/lib
    sudo ldconfig
    sudo cp include/SaleaeDeviceApi.h /usr/local/include

## Makefile

Now change to a directory where you'll develop your application and create a
proper Makefile.  Here's my simple one that builds main.cpp and links against
Saleae's library:

    APP = analyzer
    PREFIX ?= /usr/local
    CXXFLAGS += -std=c++11 -fpic -g -Wall

    ifeq ($(shell uname -m),x86_64)
    LDFLAGS += -lSaleaeDevice64
    else
    LDFLAGS += -lSaleaeDevice
    endif

    SRCS = main.cpp

    OBJS = $(patsubst %.cpp,%.o,$(SRCS))

    %.o: %.cpp
      g++ $(CXXFLAGS) -c $< -o $@

    $(APP): $(OBJS)
      g++ $^ $(LDFLAGS) -o $@

    install: $(APP)
      install -d $(PREFIX)/bin
      install $< $(PREFIX)/bin/$<

    clean:
      @rm -f $(OBJS) $(APP)

    .PHONY: install clean

## Application

To use the Logic16, your C++ application simply needs to:

* register handlers for connection and disconnection via the
  `RegisterOnConnect()` and `RegisterOnDisconnect()` methods.  The OnConnect
  handler should use `RegisterOnReadData()` to register a handler for incoming
  data and you may want to register an error handler as well via
  `RegisterOnError()`.  Finally, the OnConnect handler should configure the
  channels you wish to sample via SetActiveChannels(), set the sample rate
  via `SetSampleRateHz()`, and use `ReadStart()` to begin recording.
* spin up a worker thread to deal with data from the logic analyzer.  A pointer
  to the incoming data buffer will be provided to your OnReadData handler,
  however that handler shouldn't process the data in place (and depending on
  how high your sampling rate is, it can be several megabytes of data).  The
  handler should instead queue that pointer up and signal to the worker thread
  to process it.  The worker should then free that memory by calling the
  provided `DeleteU8ArrayPtr()` method.
* call `BeginConnect()` to connect to a Logic16
* join the worker thread, there's nothing further to do in main()

I tried implementing the worker thread using pthread (pass -pthread as one of
the CFLAGS if you want to do that) and then used the "new" C++11
[std::thread](http://en.cppreference.com/w/cpp/thread/thread) in
its place, either way works fine.  You can use std::queue or a list or some
other approach to buffer up data for the worker thread but whatever you choose
should be thread-safe since you're dealing with an asynchronous producer and a
consumer who are using the same queue.  pthread provides mutexes and semaphores
for this and C++11 includes several concurrency schemes.

The Saleae API is a bit crufty and misses some common best practices but is
otherwise functional and well-documented in its header file.  I would like to
mention a few things:

* You may ignore the `SALEAE_DEVICE_API` and `__stdcall` cruft that you see in
  SaleaeDeviceApi.h and their sample application.
* `SetSampleRateHz()` must have a valid sample rate passes to it, otherwise the
  program proceeds and then segfaults.
* `SetActiveChannels()` takes a pointer to an array and a number of elements.
  Although it's not marked const, that array seems to be copied or otherwise
  used safely so a stack variable is acceptable.  You must use this array to
  map what channels are active (and their order in the bitmask you get back in
  the OnReadData handler).
* You'll be given a pointer to an array of U8's in the OnReadData handler.
  This however contains 16-bit samples (regardless of which channels you've
  enabled) and you should process it accordingly.  That is, if channel 0 is
  your SPI SCK, channel 1 is your MISO, channel 2 is MOSI, and channel 3 is
  ENABLE, you would have mapped something like:

    U32 channels[4] = { 0, 1, 2, 3 };

  and you'll care about the lower four bits of each 16-bit chunk of that block
  of data.  The size argument given to your OnReadData handler is in bytes and
  it may be anything (the documentation says that it's roughly a 20Hz rate of
  block retrieval, so whatever number of bytes that comes out to given your
  chosen sampling rate).
* [std::bitset](http://www.cplusplus.com/reference/bitset/bitset/) makes
  printing bits (either for debugging or as part of the output of your program)
  quite easy.

Of course with this "raw" API, you're responsible for making sense of the data,
"triggering", and so on.  You can implement that with a fairly simple state
machine (for example, look for ENABLE and SCK transitions for SPI), just be
sure to sample your data bits at the right time based on the bus in question
and its configuration -- for instance a SPI bus may be used in such a way that
data is available on the falling edge of SCK and sampled on the rising edge.
