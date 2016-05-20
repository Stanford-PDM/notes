# Folder hierarchy
TODO: explain what each folder is

```
hyperdsl
└── delite
    └── framework
        └── delite-test
        └── src
            └── ppl
                └── delite
                    └── framework
                        ├── analysis
                        ├── codegen
                        │   ├── cpp
                        │   ├── cuda
                        │   ├── delite
                        │   │   ├── generators
                        │   │   └── overrides
                        │   ├── opencl
                        │   ├── restage
                        │   └── scala
                        ├── datastructures
                        ├── extern
                        │   ├── codegen
                        │   │   ├── cpp
                        │   │   ├── cuda
                        │   │   ├── opencl
                        │   │   └── scala
                        │   └── lib
                        ├── ops
                        └── transform

```


# How to debug compile times
### To explore timings of sbt tasks
[https://github.com/jrudolph/sbt-optimizer](https://github.com/jrudolph/sbt-optimizer)

### To explore timings of scalac compiler phases
`-Dscala.timings` has been removed since scala 2.9 or 2.10 (using 2.11 even with scala-virtualized)

### To explore timings of scalac by looking at the jvm execution

#### hprof
Add this lines to */usr/local/etc/sbtopts*

```sh
# Enable hprof
-J-agentlib:hprof=cpu=samples
```

and then launch sbt normally, you can find a log of hprof in *java.hprof.txt*

TODO: potential resource http://www.javaworld.com/article/2075884/core-java/diagnose-common-runtime-problems-with-hprof.html

#### Java Mission Control / Java Flight Recorder
Add these two lines to */usr/local/etc/sbtopts*

```sh
# Enable flight recorder
-J-XX:+UnlockCommercialFeatures
-J-XX:+FlightRecorder
```

and then launch Java Mission Control (should be included in the JDK, just run `jmc`) and attach to the sbt process. The diagram and list under Code->Overview->Hot Packages seems to be most useful to guess where the compiler spends a lot of time.

see: 

- [Java Flight Recorder Guide](http://docs.oracle.com/javacomponents/jmc-5-5/jfr-runtime-guide/about.htm#JFRRT110)
- [How to use Java Mission Control (Basic)](http://hirt.se/blog/?p=364)
- [How to use Java Mission Control (Advanced)](http://hirt.se/blog/?p=370)

## Cool reources
https://gist.github.com/retronym/86ec6ad9ccd2c22f6148