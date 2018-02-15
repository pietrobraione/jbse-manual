# Introduction

Welcome. Let me introduce you to JBSE and explain what JBSE is and what it can do.

JBSE is a special-purpose Java Virtual Machine. As you may know, a Java Virtual Machine \(JVM\) is what is necessary to execute a program written in the programming language Java. To be more precise, a JVM interprets the special format emitted by the Java compiler. This translates a Java program into a lower level language, the Java bytecode, that a JVM must interpret and execute. Java achieves its portability across different platforms because, differently from machine language, the Java bytecode is platform-independent: It is sufficient to port a JVM implementation across different platforms, and automatically all the Java programs compiled to bytecode can be executed on all these platforms without changes. The Java bytecode format is precisely documented in the Java Virtual Machine Specification \(JVMS\) books, that describe how a compliant JVM must execute a program in Java bytecode. The reference JVM implementation is Oracle's Hotspot JVM, but there are many other ones, e.g., IBM's J9 or aicas' JamaicaVM.

So JBSE is a JVM, and therefore it is used to execute Java software. Right?

Well, not really.

JBSE's main purpose is to _analyze_, rather than _execute_, Java software.

## Software analysis

Let us face the reality: Too often software systems do not work as expected. There are many reasons why this happens, but perhaps the most cogent is that software systems quickly turn complex, and when they turn complex, they usually turn _extremely_ complex. The Windows 10 operating system, for example, is about 50 millions of lines of code over 3.5 millions of files, and when checked out it occupies about 300 GB of disk space. Complexity in structure implies complexity in behavior, and unforeseeable behaviors are the root cause of bugs. A possible way to dominate this complexity is to empower the software engineer with tools that help him or her with understanding how the system behaves. These tools perform what is commonly called _software analysis_ and can be roughly classified into two categories: _static_ and _dynamic_ analysis tools. The well known [Findbugs](http://findbugs.sourceforge.net/), [Checkstyle](http://checkstyle.sourceforge.net/) and [PMD](http://pmd.sourceforge.net/) tools are examples of tools that perform a static analysis, that is, that extract information about a software system without executing it. The three cited tools scan the source code in search for the occurrence of a number of predefined code patterns, each indicating a different kind of bug. On converse, dynamic analysis tools gather information on the software under analysis by observing the effects of its execution. Testing is the most essential dynamic analysis activity, that is based upon observing the effects of the execution of the system fed by a set of test inputs in search for the manifestations of software defects.

