---
layout: post
title: "Inspecting JIT compiled assembly code in Oracle JVM"
description: ""
category: 
tags: ["java", "jvm"]
---
{% include JB/setup %}


JVM is one of the most well engineered piece of software. Most popular implementations (openJDK / Oracle) are bundled with a JIT compiler called "HotSpot", which identifies "hotspots" in the runtime as the bytecode is executing and compiles them to native code to get native performance. To summarize the effects of a JIT compiler on JVM, Lets look at an example

{% gist 1b05220a5815305a8902 %}

This program does a reduce operation iterating over 100 million times. Running this program with JIT enabled (default) and disabled would show us the importance of JIT compiler.

    shell-> javac Main.java
    shell-> java Main # JIT Enabled by default
    Result : 5000000050000000
    Time taken : 30.757 ms
    shell-> java -Djava.compiler=NONE Main # JIT disabled
    Result : 5000000050000000
    Time taken : 1582.021 ms

JIT compiled version ran atleast 50x faster than the other. As we are convinced of the speedups we get from JIT, we might be wondering about the assembly code it generated during the runtime. We might need to inspect this assembly code for various reasons like

* Understanding JIT compiler
* Diagnosing performance issues
* Making code JIT friendly etc.. 

Inspecting the JIT compiled assembly code can be done using the flag "XX:+PrintAssembly". We have to turn on the VM diagnostics for this flag to work. As this option is also dependent on the [basic disassembler for hotspot](https://kenai.com/projects/base-hsdis), we need to download it and place it in our LD_LIBRARY_PATH before invoking the JVM with these flags. 
    
    shell->#Downloading disassembler for MacOS
    shell->sudo wget -O  /usr/lib/hsdis-amd64.dylib \
    https://kenai.com/projects/base-hsdis/downloads/download/gnu-versions/hsdis-amd64.dylib

    shell-> java -server -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly Main
    Java HotSpot(TM) 64-Bit Server VM warning: PrintAssembly is enabled;
     turning on DebugNonSafepoints to gain additional output
    Loaded disassembler from hsdis-amd64.dylib
    Decoding compiled method 0x0000000104e77d50:
    Code:
    [Disassembling for mach='i386:x86-64']
    [Entry Point]
    [Constants]
      # {method} {0x000000011deaefc8} 'hashCode' '()I' in 'java/lang/String'
      #           [sp+0x40]  (sp of caller)
      0x0000000104e77ec0: mov    0x8(%rsi),%r10d
      0x0000000104e77ec4: shl    $0x3,%r10

      .... [Trimmed for readability]

Many times while debugging performance issues, we focus on a set of methods and filtering the disassembler output only for these methods can help us analyze effectively. This can be done with the help of "-XX:CompileCommand" flag as shown below.

    shell-> java -server -XX:+UnlockDiagnosticVMOptions -XX:CompileCommand=print,*Main.seriousLoop Main
    CompilerOracle: print *Main.seriousLoop
    Java HotSpot(TM) 64-Bit Server VM warning: printing of assembly code is enabled;
    turning on DebugNonSafepoints to gain additional output
    Compiled method (c1)      72   46 %     3       Main::seriousLoop @ 4 (25 bytes)
     total in heap  [0x0000000102dde4d0,0x0000000102dde968] = 1176
     relocation     [0x0000000102dde5f0,0x0000000102dde628] = 56
     main code      [0x0000000102dde640,0x0000000102dde7e0] = 416
     stub code      [0x0000000102dde7e0,0x0000000102dde870] = 144
     oops           [0x0000000102dde870,0x0000000102dde878] = 8
     metadata       [0x0000000102dde878,0x0000000102dde880] = 8
     scopes data    [0x0000000102dde880,0x0000000102dde8c0] = 64
     scopes pcs     [0x0000000102dde8c0,0x0000000102dde960] = 160
     dependencies   [0x0000000102dde960,0x0000000102dde968] = 8
    Loaded disassembler from hsdis-amd64.dylib
    Decoding compiled method 0x0000000102dde4d0:
    Code:
    [Disassembling for mach='i386:x86-64']
    [Entry Point]
    [Verified Entry Point]
    [Constants]
      # {method} {0x000000011c1ea3d0} 'seriousLoop' '()J' in 'Main'
      0x0000000102dde640: mov    %eax,-0x14000(%rsp)
      0x0000000102dde647: push   %rbp
      0x0000000102dde648: sub    $0x40,%rsp
      0x0000000102dde64c: movabs $0x11c1ea6c8,%rsi  
        ; {metadata(method data for {method} {0x000000011c1ea3d0} 'seriousLoop' '()J' in 'Main')}
      0x0000000102dde656: mov    0x64(%rsi),%edi
      0x0000000102dde659: add    $0x8,%edi
      0x0000000102dde65c: mov    %edi,0x64(%rsi)
      0x0000000102dde65f: movabs $0x11c1ea3d0,%rsi  
        ; {metadata({method} {0x000000011c1ea3d0} 'seriousLoop' '()J' in 'Main')}
      0x0000000102dde669: and    $0x1ff8,%edi
      0x0000000102dde66f: cmp    $0x0,%edi
      0x0000000102dde672: je     0x0000000102dde776  ;*lconst_0
                                                    ; - Main::seriousLoop@0 (line 4)

    .... [Trimmed for readability]

As we can see, the generated assembly code also includes details like line number, function name, class etc.. which helps us to understand the code flow. Full disassembler output is available [here](https://gist.github.com/varadharajan/eeea5fc53495401e1145)

Reference: [PrintAssembly Manual](https://wikis.oracle.com/display/HotSpotInternals/PrintAssembly)