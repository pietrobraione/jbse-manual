# Introduction

Welcome. Let me introduce you to JBSE and explain what it is and what it can do.

JBSE is a special-purpose Java Virtual Machine. As you may know, a Java Virtual Machine \(JVM\) is what is necessary to execute a program written in the programming language Java. To be more precise, a JVM interprets the special format emitted by the Java compiler, the Java bytecode. Java achieves its portability across different platforms because its compiler does not translate Java programs to machine language, but instead it translates it into the Java bytecode lower level language that a JVM must interpret and execute. The advantage of this approach is that the Java bytecode is platform-independent: It is sufficient to port a JVM implementation across different platforms, and automatically all the Java programs compiled to bytecode can be executed on all these platforms without changes. The Java bytecode format is precisely documented in the Java Virtual Machine Specification \(JVMS\) books, that describe how a compliant JVM must execute a program in Java bytecode. The reference JVM implementation is Oracle's Hotspot JVM, but there are many other ones, e.g., IBM's J9 or aicas' JamaicaVM.

So JBSE is a JVM, and therefore it is used to execute Java software. Right?

Well, not really.

JBSE's main purpose is to _analyze_, rather than _execute_, Java software.

## Software analysis

Let us face the reality: Too often software systems do not work as expected. There are many reasons why this happens, but the possibly most cogent one is that software systems quickly turn complex, and when they turn complex, they usually turn _extremely_ complex. The Windows 10 operating system, for example, is about 50 millions of lines of code over 3.5 millions of files, and when checked out it occupies about 300 GB of disk space. Complexity in structure implies complexity in behavior, and unforeseeable behaviors are the root cause of bugs. A possible way to dominate this complexity is to empower the software engineer with tools that help him or her with understanding how the system behaves. These tools perform what is commonly called _software analysis_ and can be roughly classified into two categories: _static_ and _dynamic_ analysis tools. Static analysis tools extract information about a software system without executing it. The well known [Findbugs](http://findbugs.sourceforge.net/), [Checkstyle](http://checkstyle.sourceforge.net/) and [PMD](http://pmd.sourceforge.net/) tools perform a kind of static analysis, that is, they scan the source code of the system in search for the occurrence of a number of predefined code patterns, each indicating the possible occurrence of a different kind of bug. Static analysis techniques usually require the source code of the software and produce approximate answers. On converse, dynamic analysis tools gather information on the software under analysis by observing the effects of its execution. Testing is the quintessential dynamic analysis activity: It observes the effects of the execution of the system software when fed by a set of test inputs, in search for the manifestations of software defects. Dynamic analyses are usually very precise by their same nature, but bound to the \(few\) executions they are able to observe, they usually produce less general results than static analyses: For example, as observed by Dijkstra, testing alone cannot be used to assess the _absence_ of some kind of software bugs.

JBSE performs a kind of dynamic analysis that is called _symbolic execution_, that is amenable both to verify the correctness of a program with respect to some desired properties expressed as assertions, and to generate test vectors for the program.

## What is symbolic execution?

If you do not know what "symbolic execution" is, then you may have a look at the corresponding [Wikipedia article](http://en.wikipedia.org/wiki/Symbolic_execution) or to some [textbook](http://ix.cs.uoregon.edu/~michal/book/). But if you are really impatient, here is a very short tutorial.

To explain what symbolic execution is we can consider that symbolic execution is to testing what symbolic equation solving is to numeric equation solving. Let us consider, for instance, the equation $$x^2 - 2 \cdot x + 1  = 0$$, of which we want to find its real solutions. This second degree equation is numeric, meaning that all its coefficients are numbers. According to the value of the discriminant $$\Delta$$ the equation can have two real solutions \(this happens when $$\Delta > 0$$\), one real solution \(when $$\Delta = 0$$\) or no real solution \(when $$\Delta < 0$$\). In this case the equation has one real solution being $$\Delta = (-2)^2 - 4 \cdot 1 \cdot 1 = 4 - 4 = 0$$. Conversely, the equation $$x^2 - b \cdot x + 1 = 0$$ is symbolic, because one of the coefficients $$b$$ is not a number but a _symbol_, standing for an unknown numeric value ranging in a \(possibly infinite\) set of admissible values. If we assume that this set is the set the real numbers, then the discriminant of the second equation is $$\Delta = b^2 - 4$$. As with the numeric equations, to determine the solution of the symbolic equation we need to split cases based on the sign of the discriminant. But differently from , where exactly one case holds, symbolic equation solving may require to follow more than one of them. Depending on the possible values of $$b$$ our example symbolic equation may fall in one of three cases: If $$|b| > 2$$ the discriminant is greater than zero and the equation has two real solutions, if $$b = 2$$ or $$b = -2$$ the discriminant is zero and the equation has one real solution. Finally, if $$-2 < b < 2$$, the discriminant is less than zero and the equation has no real solutions. Since all the three subsets for $$b$$ are nonempty any of the three cases may hold. As a consequence, the solution of a symbolic equation is usually expressed as a set of _summaries_. A summary associates a condition on the symbolic parameters with a corresponding possible result of the equation, where the result can be a number \_or \_an expression in the symbols. For our running example the solution produces as summaries $$|b| > 2 \rightarrow x = (b + \sqrt{b^2 - 4})/2$$, $$|b| > 2 \rightarrow x = -(b + \sqrt{b^2 - 4})/2$$, $$b = 2 \rightarrow x = 1$$, and $$b = -2 \rightarrow x = -1$$. Note that summaries overlap where a combination of parameters values \($$|b| > 2$$ in the previous case\) yield multiple results, and that the union of the summaries does not span the whole domain for $$b$$, because some values for $$b$$ yield no result.

Symbolic execution is a program analysis technique that is based on performing the execution of a program with input values that may be symbols standing for sets of possible numeric \(concrete\) values. Consider for example the following Java program:

```java
package smalldemos.ifx;

public class IfExample {
    boolean a, b;
    public void m(int x) {
        if (x > 0) {
            a = true;
        } else {
            a = false;
        }
        if (x > 0) {
            b = true;
        } else {
            b = false;
        }
        assert a == b;
    }
}
```

This program is the typical "double-if" example that is customarily used to illustrate what symbolic execution is. It is the sequence of two `if` statement with same condition, where the variables involved in the conditions are never modified. This ensures that every execution of the program will execute either both `then` branches or both `else` branches. The final assertion requires for the program to be correct that the variables `a` and `b` have same final value, a fact that trivially holds given the previous premise. Let us test the method `m` with input, say, `x == 3`:

* The JVM first evaluates the branch condition `x > 0` of the first `if` statement. Being `x == 3` this yields `(3 > 0) == true`: Thus the `then` branch of the first `if` statement is selected for execution and `a` is set to `true`. Then the execution continues with the second `if` statement.
* The JVM evaluates the branch condition of the second `if` statement, that is again `x > 0`. Since the value of `x` is still `3`, the `then` branch of the second `if` statement is selected for execution and `b` is set to true. Then the execution continues with the `assert` statement.
* The JVM evaluates the condition `a == b` of the `assert` statement. Since both `a` and `b` are set to `true`, the condition holds and the method terminates correctly.

Now let us perform symbolic execution of the same method `m` with a symbolic value, say $$x_0$$, for its input `x`. We do not make any assumption on what the value of $$x_0$$ might be: It could stand for any possible `int` value. This is how JBSE executes the method:

* JBSE evaluates the branch condition `x > 0` of the first `if` statement. Since `x ==`$$x_0$$, and no assumption is made on the concrete value $$x_0$$ stands for, JBSE cannot determine what is the next statement that must be executed. Therefore JBSE does what we did in the case of the quadratic equation with symbolic coefficients: It splits cases.
* First, JBSE assumes that the branch condition `x > 0` evaluates to `true`. Being `x ==`$$x_0$$ this happens when $$x_0 > 0$$. 
  * In this case, JBSE selects for execution the `then` branch of the first `if` statement, `a` is set to `true`, and the execution continues with the second `if` statement.
  * JBSE then evaluates the second branch condition: But since it has previously assumed that $$x_0 > 0$$ the second branch condition always evaluates to `true`. JBSE selects the `then` branch of the second `if` statement, `b` is set to `true`, and the execution continues with the `assert` statement.
  * JBSE evaluate the condition `a == b` of the `assert` statement. Again, `a` and `b` are set to `true`, the condition holds and the method execution terminates correctly.
* Once finished the analysis of the case $$x_0 > 0$$ JBSE _backtracks_, i.e., restores the state of the execution where the next statement to be executed is the first `if` statement, and considers the opposite case, i.e., the case where the branch condition `x > 0` evaluates to `false`. Since in the backtracked state it is again `x ==`$$x_0$$, this happens when $$x_0 \leq 0$$.
  * Now the `else` branch of the first `if` statement is followed and `a` is set to false. 
  * The execution continues with the second if statement, and since JBSE has now assumed that $$x_0 \leq 0$$ it will evaluate the second branch condition to false. The else branch of the second if statement is followed and b is set to false.
  * Finally, JBSE executes the assert statement. Being `a` and `b` both set to `false`, the assertion condition holds and the method execution terminates correctly.

This example shows you that symbolic execution is not much different from ordinary \(concrete\) execution. The main difference is that at some point of the execution, because of the presence of symbolic values, it might be unclear what is the next thing to do. In this case it is necessary to introduce an assumption on the possible values of the symbolic inputs so the next action is unambiguously identified. Such an assumption is called the _path condition_, because it is progressively built as symbolic execution explores a path in the program. All the input values

