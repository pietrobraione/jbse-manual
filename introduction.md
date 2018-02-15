# Introduction

Welcome. Let me introduce you to JBSE and explain what JBSE is and what it can do.

JBSE is a special-purpose Java Virtual Machine. As you may know, a Java Virtual Machine \(JVM\) is what is necessary to execute a program written in the programming language Java. To be more precise, a JVM interprets the special format emitted by the Java compiler, the Java bytecode. Java achieves its portability across different platforms because its compiler does not translate Java programs to machine language, but instead it translates it into the Java bytecode lower level language that a JVM must interpret and execute. The advantage of this approach is that the Java bytecode is platform-independent: It is sufficient to port a JVM implementation across different platforms, and automatically all the Java programs compiled to bytecode can be executed on all these platforms without changes. The Java bytecode format is precisely documented in the Java Virtual Machine Specification \(JVMS\) books, that describe how a compliant JVM must execute a program in Java bytecode. The reference JVM implementation is Oracle's Hotspot JVM, but there are many other ones, e.g., IBM's J9 or aicas' JamaicaVM.

So JBSE is a JVM, and therefore it is used to execute Java software. Right?

Well, not really.

JBSE's main purpose is to _analyze_, rather than _execute_, Java software.

## Software analysis

Let us face the reality: Too often software systems do not work as expected. There are many reasons why this happens, but the possibly most cogent one is that software systems quickly turn complex, and when they turn complex, they usually turn _extremely_ complex. The Windows 10 operating system, for example, is about 50 millions of lines of code over 3.5 millions of files, and when checked out it occupies about 300 GB of disk space. Complexity in structure implies complexity in behavior, and unforeseeable behaviors are the root cause of bugs. A possible way to dominate this complexity is to empower the software engineer with tools that help him or her with understanding how the system behaves. These tools perform what is commonly called _software analysis_ and can be roughly classified into two categories: _static_ and _dynamic_ analysis tools. Static analysis tools extract information about a software system without executing it. The well known [Findbugs](http://findbugs.sourceforge.net/), [Checkstyle](http://checkstyle.sourceforge.net/) and [PMD](http://pmd.sourceforge.net/) tools perform a kind of static analysis, that is, they scan the source code of the system in search for the occurrence of a number of predefined code patterns, each indicating the possible occurrence of a different kind of bug. Static analysis techniques usually require the source code of the software and produce approximate answers. On converse, dynamic analysis tools gather information on the software under analysis by observing the effects of its execution. Testing is the quintessential dynamic analysis activity: It observes the effects of the execution of the system software when fed by a set of test inputs, in search for the manifestations of software defects. Dynamic analyses are usually very precise by their same nature, but bound to the \(few\) executions they are able to observe, they usually produce less general results than static analyses: For example, as observed by Dijkstra, testing alone cannot be used to assess the _absence_ of some kind of software bugs.

JBSE performs a kind of dynamic analysis that is called _symbolic execution_, that is amenable both to verify the correctness of a program with respect to some desired properties expressed as assertions, and to generate test vectors for the program.

## What is symbolic execution?

If you want to learn what "symbolic execution" is you can have a look at the corresponding [Wikipedia article](http://en.wikipedia.org/wiki/Symbolic_execution) or to some [textbook](http://ix.cs.uoregon.edu/~michal/book/). But if you are really impatient, symbolic execution is to testing what symbolic equation solving is to numeric equation solving. Let us consider, for instance, the equation $$x^2 - 2 \cdot x + 1  = 0$$. This second degree equation is numeric, meaning that all its coefficients are numbers. According to the value of the discriminant $$\Delta$$ the equation can have two real solutions \(this happens when $$\Delta > 0$$\), one real solution \(when $$\Delta = 0$$\) or no real solution \(when $$\Delta < 0$$\). In this case the equation has one real solution being $$\Delta = (-2)^2 - 4 \cdot 1 \cdot 1 = 4 - 4 = 0$$. Conversely, the equation $$x^2 - b \cdot x + 1 = 0$$ is symbolic, because one of the coefficients $$b$$ is not a number but a _symbol_, standing for an infinite set of numeric values. 



