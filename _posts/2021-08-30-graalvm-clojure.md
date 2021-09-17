---
layout: post
title: Native Clojure with GraalVM
description: Blazing startup time, and significantly less memory footprint
image: graalvm.png
---

## Clojure
Clojure is one of the most powerful programming languages out there. It's unique approach to functional programming, being a dynamically typed functional language is very powerful. It's a dialect of Lisp, one of oldest and most powerful languages itself. Lisp was one of the first to introduce concepts like first class functions or anonymous functions, garbage collection and others that are still used till this day.

![powerful_lisp](https://imgs.xkcd.com/comics/lisp_cycles.png)

## GraalVM
[GraalVM](https://www.graalvm.org/) is a Java virtual machine that was develpoed by the Oracle Corporation in 2019. Although it's a JVM but it supports other languages like Clojure, NodeJS, and more.

One of the very cool features of GraalVM is Ahead-of-Time (AOT) compilation, which is what we're gonna discuss here. Ahead of time compilation allows for a blazingly faster startup time and much less memory footprint.

## Building a native image of your Clojure app
Using GraalVM with Clojure is pretty easy and straightforward. Start by first downloading the GraalVM binaries for your selected Java version from https://github.com/graalvm/graalvm-ce-builds/releases, unpack it and make sure its in your system's `PATH`.
Once the GraalVM command line tools are recognized in the system, you can install `native-image` using:

`gu install native-image`

After that, make sure that your `project.clj` in your clojure project contains the `:main` and that ahead of time (AOT) compilation is enabled:

```clojure
(defproject graalvm_test "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :license {:name "EPL-2.0 OR GPL-2.0-or-later WITH Classpath-exception-2.0"
            :url "https://www.eclipse.org/legal/epl-2.0/"}
  :dependencies [[org.clojure/clojure "1.10.1"]]

  :repl-options {:init-ns graalvm-test.core})

  ;; add the main namespace
  :main graalvm-test.core

  ;; add aot to the build profile
  :profiles {:uberjar {:aot :all}}
```

Finally, you'll need to add a main function to your clojure project in the main namespace you specified.

```clojure
(defn -main 
  [& args]
  (println "Hello World"))
```

Now, in order to build a native image, we have to build an uberjar first:

`lein do clean, uberjar`

And now you can run the `native-image` command:

```
native-image --report-unsupported-elements-at-runtime \
             --initialize-at-build-time \
             --no-server \
             -jar ./target/hello-world-0.1.0-SNAPSHOT-standalone.jar \
```

You can now run your native image:

`./hello-world-0.1.0-SNAPSHOT-standalone`

And to put it in action, let's compare the time difference between running the jar vs. the native image:

```
$ time java -jar target/hello-world-0.1.0-SNAPSHOT-standalone.jar
Hello World
java -jar target/hello-world-0.1.0-SNAPSHOT-standalone.jar  2.17s user 0.20s system 181% cpu 1.309 total

$ time ./hello-world-0.1.0-SNAPSHOT-standalone
Hello World
./hello-world-0.1.0-SNAPSHOT-standalone  0.00s user 0.01s system 29% cpu 0.034 total
```

from 2.17 seconds to less than a millisecond of runtime speed, amazing!

Building a native image of your clojure projects isn't always this straightforward, though. Some clojure libraries use dynamic class loading to load some of its components, and for that you'll need to supply a [reflection](https://medium.com/swlh/reflection-a-hidden-jvm-superpower-54a5a70fef0d) configuration file to GraalVM in order for it to load the class on runtime. 

## Native Carmine: Building a native image of clojure's redis client
[Carmine](https://github.com/ptaoussanis/carmine) is a very powerful [Redis](https://redis.io/) client for Clojure. It's also one of the clojure libararies that uses dynamic class loading for some of its components (i.e: a class named `org.apache.commons.pool2.impl.EvictionPolicy`). Let's say how to build a native image of a clojure project that uses Carmine.

Let's start by generating a new clojure project:

`lein new carmine_graalvm`

We'll add the carmine dependency to `project.clj`, along with what we did initially; adding the main and enabling aot compilation:

```clojure
  :dependencies [[org.clojure/clojure "1.10.1"]
                  [com.taoensso/carmine "3.1.0"]]
  :main carmine-graalvm.core
  :profiles {:uberjar {:aot :all}}
```

Next, we'll need to add some code that communicates with the redis server we want to test. Add this to your `core.clj`:

```clojure
(ns carmine-graalvm.core       
  (:require [taoensso.carmine :as car])
  (:gen-class))                
  
(defmacro wcar* [& body] `(car/wcar {} ~@body))

(defn -main                    
  [& args]                     
  (println (wcar* (car/ping)))
  (println (wcar* (car/info "server"))))
```

This code sends the command `PING` to the redis server which replies with `PONG` and then asks redis for the server info and prints that out.

Before building the native image; let's make sure our code runs by running `lein run`. Make sure that the redis server is running in your local machine.

```
$ lein run
PONG
# Server
redis_version:6.2.5
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:ee85148efbe62cad
redis_mode:standalone
os:Linux 5.11.0-34-generic x86_64
arch_bits:64
multiplexing_api:epoll
atomicvar_api:c11-builtin
gcc_version:9.3.0
process_id:5119
process_supervised:no
run_id:b99a6e6efa94367fd92d1ec52c89c0c48d215f02
tcp_port:6379
server_time_usec:1631835822597434
uptime_in_seconds:119
uptime_in_days:0
hz:10
configured_hz:10
lru_clock:4445870
executable:/home/amarnah/workspace/carmine_graalvm/redis-server
config_file:
io_threads_active:0
```

Now we can just build the source by running: 

`lein do clean, uberjar`

and then build the native image by: 

```
native-image --report-unsupported-elements-at-runtime \
             --initialize-at-build-time \
             --no-server \
             -jar ./target/carmine_graalvm-0.1.0-SNAPSHOT-standalone.jar
```

now running `./carmine_graalvm-0.1.0-SNAPSHOT-standalone` will give an exception:

```
Exception in thread "main" java.lang.IllegalArgumentException: Unable to create org.apache.commons.pool2.impl.EvictionPolicy instance of type org.apache.commons.pool2.impl.DefaultEvictionPolicy
...
```

This is because as we mentioned, carmine uses dynamic class loading for that class. To successfully build a native image of our project, we'll need to supply a reflection configuration file that'll let GraalVM know that this class is a dynamic class and is going to be loaded in the future. Add this to a file named `reflect-config.json`:

```json
[
  {
    "name":"org.apache.commons.pool2.impl.DefaultEvictionPolicy",
    "allPublicConstructors" : true
  }
]
```

and when building the native image we'll use:

`-H:ConfigurationFileDirectories=./path/to/config/dir`

and that'll make our final build command:

```
native-image --report-unsupported-elements-at-runtime \
             --initialize-at-build-time \
             --no-server \
             -H:ConfigurationFileDirectories=./path/to/config/dir \
             -jar ./target/carmine_graalvm-0.1.0-SNAPSHOT-standalone.jar
```

Comparing the time between the two variants

```
java -jar target/carmine_graalvm-0.1.0-SNAPSHOT-standalone.jar  5.81s user 0.34s system 237% cpu 2.591 total

./carmine_graalvm-0.1.0-SNAPSHOT-standalone  0.01s user 0.01s system 106% cpu 0.014 total
```

From 5.81 seconds to less than a millisecond! You can also check the memory usage and compare it in the two processes.

GraalVM can also be used in many other languages like Ruby, NodeJS, and many more!