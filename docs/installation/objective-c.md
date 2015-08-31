---
layout: docs
title: Objective-C Quick Start
---

<h1 class="page-header">gRPC in 3 minutes (Objective-C)</h1>

This document shows you how to install and build an Objective-C client for a simple Hello World example, as used in the [Overview](/docs/index.shtml).

## Installation

To run this example you should have [Cocoapods](https://cocoapods.org/#install) installed, as well as the relevant tools to generate the client library code (and a server in another language, for testing). You can obtain the latter by following [these setup instructions](https://github.com/grpc/homebrew-grpc).

## Hello Objective-C gRPC!

Here's how to build and run the Objective-C implementation of the [Hello World](https://github.com/grpc/grpc/blob/master/examples/protos/helloworld.proto) example used in the [Overview](/docs/index.html).

The example code for this and our other examples lives in the `examples`
directory. Clone this repository to your local machine by running the
following command:


````
$ git clone https://github.com/grpc/grpc.git
```

Change your current directory to `examples/objective-c/helloworld`

````
$ cd examples/objective-c/helloworld
```

### Try it!
To try the sample app, we need a gRPC server running locally. Let's compile and run, for example, the C++ `Greeter` server:

```
$ pushd ../../cpp/helloworld
$ make
$ ./greeter_server &
$ popd
```

Now have Cocoapods generate and install the client library for our .proto files:

```
$ pod install
```

This might have to compile OpenSSL, which takes around 15 minutes if Cocoapods doesn't have it yet on your computer's cache).

Finally, open the XCode workspace created by Cocoapods, and run the app. You can check the calling code in `main.m` and see the results in XCode's log console.

The code sends a `HLWHelloRequest` containing the string "Objective-C" to a local server. The server responds with a `HLWHelloResponse`, which contains a string that is then output to the log.

## Tutorial

You can find a more detailed tutorial in [gRPC Basics: Objective-C](/docs/tutorials/basic/objective-c.html).
