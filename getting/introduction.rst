############
Introduction
############
Welcome. Let me introduce you to JBSE and explain what it is and what it can do.

JBSE is a special-purpose Java Virtual Machine. As you may know, a Java Virtual Machine (JVM) is what is necessary to execute a program written in the programming language Java. To be more precise, a JVM interprets the special format emitted by the Java compiler, the Java bytecode. Java achieves its portability across different platforms because its compiler does not translate Java programs to machine language, but to Java bytecode, that a JVM must interpret and execute. The advantage is that, differently from machine language, the Java bytecode is platform-independent: It is sufficient to port a JVM implementation across different platforms, and automatically all the Java programs compiled to bytecode can be executed unchanged on all these platforms. The Java bytecode format is precisely documented in the `Java Virtual Machine Specification (JVMS) books`_, that describe how a compliant JVM must execute a program in Java bytecode. The reference JVM implementation is Oracle's Hotspot_ JVM, but there are many other ones, e.g., IBM's J9_ or aicas' JamaicaVM_.

So JBSE is a JVM, and therefore it can be used as a drop-in replacement to Hotspot to execute Java software. Right?

Well, not really.

JBSE's main purpose is to *analyze*, rather than *execute*, Java software.

*****************
Software analysis
*****************

Let us face the reality: Too often software systems do not work as expected. There are many reasons why this happens, but the  most cogent one is possibly that software systems quickly turn complex, and when they turn complex, they usually turn *extremely* complex. The Windows 10 operating system, for example, is about 50 millions of lines of code over 3.5 millions of files, and when checked out it occupies about 300 GB of disk space. Complexity in structure implies complexity in behavior, and unforeseeable behaviors are the root cause of bugs. A possible way to dominate this complexity is to empower the software engineer with tools that help him or her with understanding how the system behaves. These tools perform what is commonly called *software analysis* and can be roughly classified into two categories: *static* and *dynamic* analysis tools. Static analysis tools extract information about a software system without executing it. The well known Findbugs_, Checkstyle_ and PMD_ tools perform a kind of static analysis based on the idea of scanning the source code of the system in search for the occurrence of a number of predefined code patterns, each indicating the possible occurrence of a different kind of bug. Static analysis techniques usually require the source code of the software and produce approximate answers, where false alarms and missed bugs are the norm, but their answers, when correct, can provide very general information on the correctness of the program. On converse, dynamic analysis tools gather information on the software under analysis by observing the effects of its execution. Testing is the quintessential dynamic analysis activity: It observes the effects of the execution of the system software when fed by a set of test inputs, in search for the manifestations of software defects. Dynamic analyses are usually very precise by their nature, but bound to the (few) executions they are able to observe, they usually produce less general results than static analyses: For example, `as observed by Dijkstra`_, testing alone cannot be used to assess the *absence* of some kind of software bugs, while static analyses, in principle, may.

JBSE performs a kind of analysis that is called *symbolic execution*, that is amenable both to verify the correctness of a program with respect to some desired properties expressed as assertions, and to generate test vectors for the program. When used for verification JBSE expects that you specify the verification properties of interest for your project as a set of *assumptions* and *assertions*. Assumptions specify the conditions that must be satisfied for an execution to be relevant. Preconditions are a typical form of assumptions, allowing e.g. to specify the range of the possible values for the program inputs. Assertions specify the conditions that must be satisfied for an execution to be correct. JBSE attempts to determine whether some input exists that satisfies all the assumptions and falsifies at least one assertion. In this regard JBSE is more similar in spirit, implementation and mode of use to tools like `Symbolic PathFinder`_, `Sireum/Kiasan`_ and JNuke_.

***************************
What is symbolic execution?
***************************

If you do not know what "symbolic execution" is, then you may have a look at the corresponding `Wikipedia article`_ or to some textbook_. But if you are really impatient, here is a very short tutorial.

To explain what symbolic execution is we can consider that symbolic execution is to testing what symbolic equation solving is to numeric equation solving. Let us consider, for instance, the equation :math:`x^2 - 2 \cdot x + 1  = 0`, of which we want to find its real solutions. This second degree equation is numeric, meaning that all its coefficients are numbers. According to the value of the discriminant :math:`\Delta` the equation can have two real solutions (this happens when :math:`\Delta > 0`), one real solution (when :math:`\Delta = 0`) or no real solution (when :math:`\Delta < 0`). In this case the equation has one real solution being :math:`\Delta = (-2)^2 - 4 \cdot 1 \cdot 1 = 4 - 4 = 0`. Conversely, the equation :math:`x^2 - b \cdot x + 1 = 0` is symbolic, because one of the coefficients :math:`b` is not a number but a *symbol*, standing for an unknown numeric value ranging in a (possibly infinite) set of admissible values. If we assume that this set is the set of the real numbers, then the discriminant of the second equation is :math:`\Delta = b^2 - 4`. As with the numeric equations, to determine the solution of the symbolic equation we need to split cases based on the sign of the discriminant. But differently from , where exactly one case holds, symbolic equation solving may require to follow more than one of them. Depending on the possible values of :math:`b` our example symbolic equation may fall in one of three cases: If :math:`|b| > 2` the discriminant is greater than zero and the equation has two real solutions, if :math:`b = 2` or :math:`b = -2` the discriminant is zero and the equation has one real solution. Finally, if :math:`-2 < b < 2`, the discriminant is less than zero and the equation has no real solutions. Since all the three subsets for :math:`b` are nonempty any of the three cases may hold. As a consequence, the solution of a symbolic equation is usually expressed as a set of *summaries*. A summary associates a condition on the symbolic parameters with a corresponding possible result of the equation, where the result can be a number *or* an expression in the symbols. For our running example the solution produces as summaries :math:`|b| > 2 \rightarrow x = (b + \sqrt{b^2 - 4})/2`, :math:`|b| > 2 \rightarrow x = -(b + \sqrt{b^2 - 4})/2`, :math:`b = 2 \rightarrow x = 1`, and :math:`b = -2 \rightarrow x = -1`. Note that summaries overlap where a combination of parameters values (:math:`|b| > 2` in the previous case) yield multiple results, and that the union of the summaries does not span the whole domain for :math:`b`, because some values for :math:`b` yield no result.

Symbolic execution is a program analysis technique that is based on performing the execution of a program with input values that may be symbols standing for sets of possible numeric (concrete) values. Consider for example the following Java program:

.. code-block:: java

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

This program is the typical "double-if" example that is customarily used to illustrate how symbolic execution works. It is a sequence of two ``if`` statement with same condition, where the variables involved in the condition are not modified through the program. This ensures that every execution of the program will execute either both the ``then`` branches or both the ``else`` branches, never a ``then``  branch and an ``else`` branch. The final assertion requires for the program to be correct that the variables ``a`` and ``b`` have same final value, a fact that trivially holds for all the possible executions. Let us test the method ``m`` with input, say, ``x == 3``:

* The JVM first evaluates the branch condition ``x > 0`` of the first ``if`` statement. Being ``x == 3`` this yields ``(3 > 0) == true``: Thus the ``then`` branch of the first ``if`` statement is selected for execution and ``a`` is set to ``true``. Then the execution continues with the second ``if`` statement.
* The JVM evaluates the branch condition of the second ``if`` statement, that is again ``x > 0``. Since the value of ``x`` is still ``3``, the ``then`` branch of the second ``if`` statement is selected for execution and ``b`` is set to true. Then the execution continues with the ``assert`` statement.
* The JVM evaluates the condition ``a == b`` of the ``assert`` statement. Since both ``a`` and ``b`` are set to ``true``, the condition holds and the method terminates correctly.

Now let us perform symbolic execution of the same method ``m`` with a symbolic value, say :math:`x_0`, for its input ``x``. We do not make any assumption on what the value of :math:`x_0` might be: It could stand for any possible ``int`` value. This is how JBSE executes the method:

* JBSE evaluates the branch condition ``x > 0`` of the first ``if`` statement. Since ``x ==``:math:`x_0`, and no assumption is made on the concrete value :math:`x_0` stands for, JBSE cannot determine what is the next statement that must be executed. Therefore JBSE does what we did in the case of the quadratic equation with symbolic coefficients: It splits cases.
* First, JBSE assumes that the branch condition ``x > 0`` evaluates to ``true``. Being ``x ==`` :math:`x_0` this happens when :math:`x_0 > 0`.

   * In this case, JBSE selects for execution the ``then`` branch of the first ``if`` statement, ``a`` is set to ``true``, and the execution continues with the second ``if`` statement.
   * JBSE then evaluates the second branch condition: But since it has previously assumed that :math:`x_0 > 0` the second branch condition always evaluates to ``true``. JBSE selects the ``then`` branch of the second ``if`` statement, ``b`` is set to ``true``, and the execution continues with the ``assert`` statement.
   * JBSE evaluate the condition ``a == b`` of the ``assert`` statement. Again, ``a`` and ``b`` are set to ``true``, the condition holds and the method execution terminates correctly.

* Once finished the analysis of the case :math:`x_0 > 0` JBSE *backtracks*, i.e., restores the state of the execution where the next statement to be executed is the first ``if`` statement, and considers the opposite case, i.e., the case where the branch condition ``x > 0`` evaluates to ``false``. Since in the backtracked state it is again ``x ==`` :math:`x_0`, this happens when :math:`x_0 \leq 0`.

   * Now the ``else`` branch of the first ``if`` statement is followed and ``a`` is set to ``false``. 
   * The execution continues with the second ``if`` statement, and since JBSE has now assumed that :math:`x_0 \leq 0` it will evaluate the second branch condition to ``false``. The ``else`` branch of the second ``if`` statement is followed and ``b`` is set to ``false``.
   * Finally, JBSE executes the ``assert`` statement. Being ``a`` and ``b`` both set to ``false``, the assertion condition holds and the method execution terminates correctly.

This example shows you that symbolic execution is not much different from ordinary (also said *concrete*) execution of programs. The main difference is that, at some point of a symbolic execution the presence of symbolic values might make unclear what is the next thing to do (which branch of the next ``if`` statement should I follow? Should I do another iteration of the ``while`` statement I am in or should I exit the loop?). In this case a symbolic executor must introduce an assumption on the possible values of the symbolic inputs so the next action is unambiguously identified. Such an assumption is called the *path condition*, because it is progressively built as symbolic execution explores a path through the branches in the control flow graph of the program. All the input values that satisfy (i.e., solve) a path condition drive the execution of the program through the path that generated the path condition. For instance, all the input values :math:`x_0` for the "double-if" program that satisfy the condition :math:`x_0 \leq 0` drive the program execution through the ``else`` branches of the two ``if`` statements. Conversely, if a path condition has no solution, then no program inputs drive the program through the corresponding path, and the path is said to be *infeasible*. For instance, the path in the "double-if" program that goes through the ``then`` branch of the first ``if`` statement and the ``else`` branch of the second ``if`` statement is :math:`x_0 > 0 \land x_0 \leq 0`, that is clearly unsatisfiable. Correspondingly, no input exists that drives the program through this path.

If a program is deterministic, i.e., it does always the same things when fed by the same inputs, then each of its possible concrete executions yields a linear sequence of states, where each state has exactly one successor. The sequence corresponds to a single path in the control flow graph of the program. On converse its possible symbolic executions yield a *symbolic execution tree*, rooted at the initial symbolic state and branching whenever a symbolic state has more than one successor because of case splitting.

.. figure:: /img/doubleif_symbolic_tree.*
   :align: center
   :scale: 40

   Symbolic execution tree for the "double-if" program.

The previous figure reports the symbolic execution tree for the "double-if" program. Circles are program states, indicating the values stored for all the variables in the program. Arrows join a state with its possible successors, and are labeled according to the next statement to be executed: If this is an assignment, the label reports the assignment, if it is a conditional, the label reports the *evaluation* of the conditional in the pre-state. The final states that pass the assertion are represented by a green tick. The figure does not show the infeasible paths, but we will often consider the case of symbolic execution trees where all the paths through the control flow graph, be them infeasible or not, are reported. We will call *static* a symbolic execution tree that reports all the static paths (either feasible or infeasible) through the program, and by contrast we will call *dynamic* a symbolic execution tree that reports only feasible paths. The next figure, for instance, reports the static symbolic execution tree of the "double-if" example program.

.. figure:: /img/doubleif_symbolic_tree_static.*
   :align: center
   :scale: 50

   Static symbolic execution tree for the "double-if" program.
	    
The red cross signifies a final state that does not pass the assertion. The path condition for a certain path is obtained by visiting the symbolic execution tree from the root through the path, and conjoining all the edge labels for conditional expressions evaluations.

.. figure:: /img/doubleif_symbolic_tree_path.*
   :align: center
   :scale: 50

   A path in the "double-if" program and its path condition.

In the previous figure the path condition for the path marked with the red dashed arrow is given by conjoining the two expressions circled in red, yielding :math:`x_0 > 0 \land x_0 \leq 0`. Being the path condition unsatisfiable, the path is infeasible.

The "double-if" program has a finite symbolic execution tree, but this is not the general case. If the program has loops the static symbolic execution tree is infinite, and most likely also the dynamic symbolic execution tree is. Consider for instance the following program:

.. code-block:: java

   package smalldemos.loop;

   public class LoopExample {
       public void m(int n) {
           while (n > 0) {
	       --n;
	   }
	   assert n <= 0;
       }
    }

Its static symbolic execution tree is reported in the next figure:

.. figure:: /img/loop_symbolic_tree.*
   :align: center
   :scale: 40

   Static symbolic execution tree for the loop program.

If a program may diverge, i.e., it has at least one (concrete) execution that does not terminate, then this execution is infinite, and correspondingly there is an infinite path in the static symbolic execution tree for it. Note however that the vice versa does not in general hold: If the static symbolic execution tree has an infinite path, this does not necessarily imply that the program may diverge. The example loop program illustrates that: Its static symbolic execution tree has one infinite path:

.. figure:: /img/loop_symbolic_tree_path.*
   :align: center
   :scale: 40

   Infinite path in the static symbolic execution tree for the loop program.

but since we can easily prove that the program always terminates, the path is infeasible. Note also that it is not possible to exclude this path from the tree without excluding some feasible paths: In other words, it is not possible to build a dynamic symbolic execution tree containing exactly all the feasible paths.

To summarize, the symbolic execution of a program with loops may not terminate, as the symbolic executor may get stuck analyzing an infinite path, or an infinite set of finite paths. For this reason symbolic executors allow users to set an analysis budget (maximum time, maximum depth), and when they exhaust the budget then they abort the analysis. Although a symbolic executor is able in practice to analyze only a finite subset of its possible symbolic paths, consider that each symbolic path stands for a potentially infinite set of concrete paths. For this reason symbolic execution remains a technique more powerful than testing.

*****************************************
Symbolic execution with objects as inputs
*****************************************

When the inputs to a program are numeric, this is pretty much what one needs to know about symbolic execution. Things become more complex when one allows programs to take objects as inputs:

.. code-block:: java

   package smalldemos.node;

   public class NodeExample {
       public void m(Node node) {
           node.swap();
       }
    }

What if we symbolically execute the ``m`` method? As usual the value of the parameter variable ``node`` is unknown at the beginning of the execution, and the variable is initialized with a symbol :math:`node_0` standing for the unknown value. This symbol is a *symbolic reference*, and in absence of assumptions it may stand for any possible reference, either valid or invalid (``null``), to the heap memory at the initial state of the execution.

Now what if the class ``Node`` is abstract and has :math:`N` concrete subclasses, each implementing a different version of the ``swap`` method? When JBSE arrives at the ``node.swap()`` statement, to determine what is the next statement to be executed it must split cases. The possible cases are, at least, :math:`N + 1`: One case where ``node == null`` and the next statement will be the ``catch`` block, if present, for the ``NullPointerException`` that the ``node.swap()`` statement execution raises, plus the :math:`N` cases where :math:`node_0` is a reference to each of the different concrete subclasses of ``Node``, and the next statement will be the first statement of the implementation of ``node`` in the assumed class. This differs from the case where only numeric symbolic values were present, and the number of possible cases at a branch is either one or two. On can conjecture that the presence of symbolic references may cause an explosion in the number of paths in the symbolic execution tree, and indeed it is often the case.

We have said that symbolic execution at the ``node.swap()`` statement must consider *at least* :math:`N + 1` cases: Should it consider more cases for the analysis to be complete? The answer to this question is found in `this paper`_, which introduces a technique, called "lazy initialization", that is used in JBSE to resolve the use of a symbolic reference at a certain point of the symbolic execution. According to this technique the cases to be considered are the following ones:

* The symbolic reference may be ``null``, or
* The symbolic reference may be a reference to a type-compatible object assumed by lazy initialization earlier during the execution, for all :math:`K` such objects, or
* The symbolic reference may be a reference to a type-compatible object different from all the objects assumed by lazy initialization earlier during the execution, for all :math:`N` compatible types.



.. _Java Virtual Machine Specification (JVMS) books: https://docs.oracle.com/javase/specs/
.. _Hotspot: http://www.oracle.com/technetwork/java/javase/downloads/index.html
.. _J9: http://www.eclipse.org/openj9/
.. _JamaicaVM: https://www.aicas.com/cms/en/JamaicaVM
.. _Findbugs: http://findbugs.sourceforge.net/
.. _Checkstyle: http://checkstyle.sourceforge.net/
.. _PMD: http://pmd.sourceforge.net/
.. _as observed by Dijkstra: https://www.cs.utexas.edu/users/EWD/ewd02xx/EWD249.PDF
.. _Symbolic PathFinder: http://babelfish.arc.nasa.gov/trac/jpf/wiki/projects/jpf-symbc
.. _Sireum/Kiasan: http://www.sireum.org/
.. _JNuke: http://fmv.jku.at/jnuke/
.. _Wikipedia article: http://en.wikipedia.org/wiki/Symbolic_execution
.. _textbook: http://ix.cs.uoregon.edu/~michal/book/
.. _this paper: https://doi.org/10.1007/3-540-36577-X_40
