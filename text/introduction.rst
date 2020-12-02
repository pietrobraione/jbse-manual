############
Introduction
############
Welcome. Let me introduce you to JBSE and explain what it is and what it can do.

JBSE is a special-purpose Java Virtual Machine (JVM). As you may know, a JVM is what is necessary to execute a program written in the programming languages Java, Scala, Clojure, Groovy, and many others. To be more precise, a JVM is able to execute the special format emitted by the compilers of these languages, the so-called Java bytecode. The programming languages that compile to Java bytecode achieve their portability across different platforms because their compilers does not translate programs directly to machine language, but to Java bytecode that, differently from machine language, is CPU- and OS-independent. It is sufficient to port a JVM implementation across different platforms, and automatically all the programs compiled to Java bytecode can be executed unchanged on all of them. The Java bytecode format is precisely documented in the `Java Virtual Machine Specification (JVMS) books`_, that describe how a compliant JVM must execute a program in Java bytecode. The reference JVM implementation is Oracle's Hotspot_ JVM, but there are many other ones, e.g., IBM's OpenJ9_ or aicas' JamaicaVM_.

So JBSE is a JVM, and therefore it can be used as a drop-in replacement to Hotspot to execute Java (or Scala, Clojure, Groovy...) software. Right?

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

This program is the customary "double-if" example that is often used to illustrate how symbolic execution works. It is a sequence of two ``if`` statement with same condition, where the variables involved in the condition are not modified through the program. This ensures that every execution of the program will execute either both the ``then`` branches or both the ``else`` branches, never a ``then``  branch and an ``else`` branch. The final assertion requires for the program to be correct that the variables ``a`` and ``b`` have same final value, a fact that trivially holds for all the possible executions. Let us test the method ``m`` with input, say, ``x == 3``:

* The JVM first evaluates the branch condition ``x > 0`` of the first ``if`` statement. Being ``x == 3`` this yields ``(3 > 0) == true``: Thus the ``then`` branch of the first ``if`` statement is selected for execution and ``a`` is set to ``true``. Then the execution continues with the second ``if`` statement.
* The JVM evaluates the branch condition of the second ``if`` statement, that is again ``x > 0``. Since the value of ``x`` is still ``3``, the ``then`` branch of the second ``if`` statement is selected for execution and ``b`` is set to true. Then the execution continues with the ``assert`` statement.
* The JVM evaluates the condition ``a == b`` of the ``assert`` statement. Since both ``a`` and ``b`` are set to ``true``, the condition holds and the method terminates correctly.

Now let us perform symbolic execution of the same method ``m`` with a symbolic value, say :math:`x_0`, for its input ``x``. We do not make any assumption on what the value of :math:`x_0` might be: It could stand for any possible ``int`` value. This is how JBSE executes the method:

* JBSE evaluates the branch condition ``x > 0`` of the first ``if`` statement. Since ``x ==`` :math:`x_0`, and no assumption is made on the concrete value :math:`x_0` stands for, JBSE cannot determine what is the next statement that must be executed. Therefore JBSE does what we did in the case of the quadratic equation with symbolic coefficients: It splits cases.
* First, JBSE assumes that the branch condition ``x > 0`` evaluates to ``true``. Being ``x ==`` :math:`x_0` this happens when :math:`x_0 > 0`.

   * In this case, JBSE selects for execution the ``then`` branch of the first ``if`` statement, ``a`` is set to ``true``, and the execution continues with the second ``if`` statement.
   * JBSE then evaluates the second branch condition: But since it has previously assumed that :math:`x_0 > 0` the second branch condition always evaluates to ``true``. JBSE selects the ``then`` branch of the second ``if`` statement, ``b`` is set to ``true``, and the execution continues with the ``assert`` statement.
   * JBSE evaluate the condition ``a == b`` of the ``assert`` statement. Again, ``a`` and ``b`` are set to ``true``, the condition holds and the method execution terminates correctly.

* Once finished the analysis of the case :math:`x_0 > 0` JBSE *backtracks*, i.e., restores the state of the execution where the next statement to be executed is the first ``if`` statement, and considers the opposite case, i.e., the case where the branch condition ``x > 0`` evaluates to ``false``. Since in the backtracked state it is again ``x ==`` :math:`x_0`, this happens when :math:`x_0 \leq 0`.

   * Now the ``else`` branch of the first ``if`` statement is followed and ``a`` is set to ``false``. 
   * The execution continues with the second ``if`` statement, and since JBSE has now assumed that :math:`x_0 \leq 0` it will evaluate the second branch condition to ``false``. The ``else`` branch of the second ``if`` statement is followed and ``b`` is set to ``false``.
   * Finally, JBSE executes the ``assert`` statement. Being ``a`` and ``b`` both set to ``false``, the assertion condition holds and the method execution terminates correctly.

This example shows you that symbolic execution is not much different from ordinary (also said *concrete*) execution of programs. The main difference is that, at some point of a symbolic execution the presence of symbolic values might make unclear what is the next thing to do (which branch of the next ``if`` statement should I follow? Should I do another iteration of the ``while`` statement I am in or should I exit the loop?). In this case a symbolic executor must introduce an assumption on the possible values of the symbolic inputs so the next action is unambiguously identified. The sequence of all the assumptions introduced during the execution is called the *path condition*, because it is determined by the path followed by the execution through the branches in the control flow graph of the program. All the input values that satisfy (i.e., solve) a path condition drive the execution of the program through the path that generated the path condition. For instance, all the input values :math:`x_0` for the "double-if" program that satisfy the condition :math:`x_0 \leq 0` drive the program execution through the ``else`` branches of the two ``if`` statements. Conversely, if a path condition has no solution, then no program inputs drive the program through the corresponding path, and the path is said to be *infeasible*. For instance, the path in the "double-if" program that goes through the ``then`` branch of the first ``if`` statement and the ``else`` branch of the second ``if`` statement is :math:`x_0 > 0 \land x_0 \leq 0`, that is clearly unsatisfiable. Correspondingly, no input exists that drives the program through this path.

If a program is deterministic, i.e., it does always the same things when fed by the same inputs, then each of its possible concrete executions yields a linear sequence of states, where each state has exactly one successor. The sequence corresponds to a single path in the control flow graph of the program. On converse its possible symbolic executions yield a *symbolic execution tree*, rooted at the initial symbolic state and branching whenever a symbolic state has more than one successor because of case splitting.

.. _doubleif-symbolic-tree:

.. figure:: /img/doubleif_symbolic_tree.*
   :align: center
   :width: 250 px

   Symbolic execution tree for the "double-if" program.

:num:`Figure #doubleif-symbolic-tree` reports the symbolic execution tree for the "double-if" program. Circles are program states, indicating the values stored for all the variables in the program. Arrows join a state with its possible successors, and are labeled according to the next statement to be executed: If this is an assignment, the label reports the assignment, if it is a conditional, the label reports the *evaluation* of the conditional in the pre-state. The final states that pass the assertion are represented by a green tick. The figure does not show the infeasible paths, but we will often consider the case of symbolic execution trees where all the paths through the control flow graph, be them infeasible or not, are reported. We will call *static* a symbolic execution tree that reports all the static paths (either feasible or infeasible) through the program, and by contrast we will call *dynamic* a symbolic execution tree that reports only feasible paths. :num:`Figure #doubleif-symbolic-tree-static`, for instance, reports the static symbolic execution tree of the "double-if" example program. The red cross signifies a final state that does not pass the assertion. The path condition for a certain path is obtained by visiting the symbolic execution tree from the root through the path, and conjoining all the edge labels for conditional expressions evaluations.

.. _doubleif-symbolic-tree-static:

.. figure:: /img/doubleif_symbolic_tree_static.*
   :align: center
   :width: 600 px

   Static symbolic execution tree for the "double-if" program.

In :num:`Figure #doubleif-symbolic-tree-path` the path marked with the red dashed arrow has as path condition the logical "and" of the two expressions surrounded by a red circle, i.e., :math:`x_0 > 0 \land x_0 \leq 0`. Being the path condition unsatisfiable, the path is infeasible.

.. _doubleif-symbolic-tree-path:

.. figure:: /img/doubleif_symbolic_tree_path.*
   :align: center
   :width: 600 px

   A path in the "double-if" program and its path condition.

The "double-if" program has a finite symbolic execution tree, but this is not the case in general. If the program has loops the static symbolic execution tree is infinite, and most likely also the dynamic symbolic execution tree is. Consider for instance the following program:

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

.. _loop-symbolic-tree:

.. figure:: /img/loop_symbolic_tree.*
   :align: center
   :width: 400 px

   Static symbolic execution tree for the loop program.

Its static symbolic execution tree, reported in :num:`Figure #loop-symbolic-tree`, is clearly infinite. If a program may diverge, i.e., it has at least one (concrete) execution that does not terminate, then this execution is infinite, and correspondingly there is an infinite path in the static symbolic execution tree for it. Note however that the vice versa does not in general hold: If the static symbolic execution tree has an infinite path, this does not necessarily imply that the program may diverge. The example loop program illustrates that: Its static symbolic execution tree has one infinite path, highlighted in :num:`Figure #loop-symbolic-tree-path` with a red dashed arrow, but since we can easily prove that the program always terminates, the path is infeasible. Note also that it is not possible to exclude this path from the tree without excluding some feasible paths: In other words, it is not possible to build a dynamic symbolic execution tree that exactly contains all the feasible paths.

.. _loop-symbolic-tree-path:

.. figure:: /img/loop_symbolic_tree_path.*
   :align: center
   :width: 400 px

   Infinite path in the static symbolic execution tree for the loop program.



To summarize, the symbolic execution of a program with loops may not terminate, as the symbolic executor may get stuck analyzing an infinite path, or an infinite set of finite paths. For this reason symbolic executors allow users to set an analysis budget (maximum time, maximum depth), and when they exhaust the budget then they abort the analysis. Although a symbolic executor is able in practice to analyze only a finite subset of its possible symbolic paths, consider that each symbolic path stands for a potentially infinite set of concrete paths. For this reason symbolic execution is a program analysis technique more powerful than testing.

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

Now what if the class ``Node`` is abstract and has :math:`N` concrete subclasses, each implementing a different version of the ``swap`` method? When JBSE arrives at the ``node.swap()`` statement, to determine what is the next statement to be executed it must split cases. The possible cases are, at least, :math:`N + 1`: One case where ``node == null`` and the next statement will be the ``catch`` block, if present, for the ``NullPointerException`` that the ``node.swap()`` statement execution raises, plus the :math:`N` cases where :math:`node_0` is a reference to each of the different concrete subclasses of ``Node``, and the next statement will be the first statement of the implementation of ``swap()`` in the assumed class. This differs from the case where only numeric symbolic values were present, and the number of possible cases at a branch is at most two. The situation is actually worse than this, and symbolic execution typically need to consider many more subcases than :math:`N + 1`. How many? The answer to this question is found in `this paper`_, which introduces a technique, called "lazy initialization", that is the one used by JBSE to determine which cases need to be analyzed when using a symbolic reference. According to the lazy initialization technique symbolic execution needs to consider the following cases:

* The symbolic reference may be ``null``, or
* The symbolic reference may be a reference to a *fresh* type-compatible object,  for all :math:`N` compatible types, or
* The symbolic reference may be a reference to a *non-fresh* type-compatible object, where with "non-fresh" we mean assumed by lazy initialization earlier during the execution, for all :math:`K` such objects.

Now some terminology. We will say that a symbolic reference on which symbolic execution did no assumption is *unresolved*, a symbolic reference that is assumed to be ``null`` is *resolved by null*, a symbolic reference that is assumed to refer a fresh object to be *resolved by expansion*, and a symbolic reference that is assumed to refer a non-fresh object to be *resolved by alias*.

To clarify how lazy initialization works we will now consider the following example program, that scans a list of integers and returns the sum of the stored values:

.. code-block:: java

   package esecfse2013;

   public class Target {
       int sum(List<Integer> list) {
           int tot = 0;
	   for (int item : list) {
	       tot += item;
	   }
	   return tot;
       }
   }

Let us suppose that ``List`` is an abstract class or interface whose only concrete subclass is a ``LinkedList`` class defined as follows:

.. code-block:: java

   public class LinkedList<I> {
       private Node head;

       private static class Node {
           private I value;
           private Node next;
           ...
       }
       ...
   }

Back to the ``Target.sum()`` method, to scan the input list the ``for`` loop must first access ``list`` itself, then ``list.head``, then ``list.head.next``, then ``list.head.next.next``... and so on, until the list termination criterion is met (for ``LinkedList`` data structures we will consider the case where they are ``null``-terminated). Symbolic execution of ``Target.sum()`` will initially need to determine whether the symbolic reference stored in ``list``, say  :math:`l_0`, points to an object or not. The following cases may hold: 

1. Either :math:`l_0` ``== null``,
2. Or :math:`l_0` ``!= null``, i.e., refers some object of class ``LinkedList``.

No more cases need to be considered, since ``List`` has only one concrete subclass. In case 1, the method raises a ``NullPointerException``. In case 2, the method starts iterating through the nodes of the list, and accesses ``list.head``. Since no assumption is made on what the fields of the ``list`` object store, JBSE assumes that ``list.head`` stores an unresolved symbolic reference, say :math:`n_0`. Because of this access two subcases arise:

2. :math:`l_0` ``!= null``:

  1. Either :math:`n_0` ``== null``,
  2. Or :math:`n_0` ``!= null``, i.e., refers some object of class ``LinkedList.Node``.

In case 2.1 (empty list) the method stops iterating and return the value of the ``tot`` variable, i.e., its initialization value 0. In case 2.2 the method adds to ``tot`` the content of the ``value`` field of the :math:`n_0` object. But we did not any assumption about the :math:`n_0` object itself but that it exists, so what should its ``value`` field contain? In lack of any assumption we can only say that this field contains a value constrained by its static type, thus an ``int`` value, and represent this unknown value with a symbol, say, :math:`v_0` [1]_. Then the method performs another iteration of the loop body by accessing ``list.head.next``, that stores another symbolic reference :math:`n_1`. This time *three* cases may arise:

2. :math:`l_0` ``!= null``:

  2. :math:`n_0` ``!= null``:

    1. Either :math:`n_1` ``== null``,
    2. Or :math:`n_1` ``==`` :math:`n_0`,
    3. Or :math:`n_1` refers some object of class ``LinkedList.Node`` different from :math:`n_0`.

In case 2.2.1 (list with one element) the method stops iterating and returns the value of ``tot``, that is, :math:`v_0`. In case 2.2.2 (:math:`n_1` is non-fresh) the method will diverge by iterating undefinitely through ``list.head.next.next == list.head.next.next.next == ... ==`` :math:`n_0`, never returning to the caller. In case 2.2.3 (:math:`n_1` fresh) the method will add ``list.head.next.value`` (say, :math:`v_1`) to ``tot``, and iterate once again the loop body by accessing ``list.head.next.next``, that stores yet another symbolic reference :math:`n_2`. This access yields *four* subcases:

2. :math:`l_0` ``!= null``:

  2. :math:`n_0` ``!= null``:

    3. :math:`n_1` ``!= null`` and fresh:

      1. Either :math:`n_2` ``== null``,
      2. Or :math:`n_2` ``==`` :math:`n_0`,
      3. Or :math:`n_2` ``==`` :math:`n_1`,
      4. Or :math:`n_2` refers some object of class ``LinkedList.Node`` different from :math:`n_0` and :math:`n_1`.

Case 2.2.3.1 is the case of a list with exactly two elements: The method stops iterating and returns :math:`v_0 + v_1`. Cases 2.2.3.2 and 2.2.3.3 are similar to case 2.2.2: The :math:`n_2` symbolic reference is non-fresh, and the method diverges by iterating indefinitely through the chain of ``...next...`` references. Case 2.2.3.4 is similar to case 2.2.3: The :math:`n_2` symbolic reference is fresh, and the method adds the content of the ``value`` field (say, :math:`v_2`) of the fresh object to ``tot`` and iterates once more the loop by accessing ``list.head.next.next.next``.

.. _scanlist-symbolic-tree:

.. figure:: /img/scanlist_symbolic_tree.*
   :align: center
   :width: 700 px

   Symbolic execution tree for the list scanning program.

:num:`Figure #scanlist-symbolic-tree` represents the portion of the symbolic execution tree we discussed until now. Unresolved symbolic references are depicted as arrows pointing to question marks, and resolved symbolic references are depicted as arrows pointing either to ``null`` (if the reference is resolved by ``null``) or to a heap object represented as a box (if the reference is resolved by alias or expansion). It can be easily inferred that, whenever a :math:`n_i` symbolic reference, obtained by accessing a ``list.head.next.next...next`` sequence, is used during symbolic execution, up to :math:`i + 2` cases must be considered: The case where :math:`n_i` ``== null``, the :math:`i` cases where :math:`n_i` is equal to (i.e., *aliases*) :math:`n_0`, :math:`n_1`... :math:`n_{i-1}`, and the case where :math:`n_i` points to a fresh object, different from the objects pointed by :math:`n_0, n_1, \ldots, n_{i-1}`. If we impose a bound on the number of possible objects with class ``LinkedList.Node``, say not more than :math:`W`, the symbolic execution tree will have :math:`1 + 1 + 2 + 3 + \ldots + W = 1 + \sum_{i = 1}^{W} i = 1 + W (W + 1) / 2 \in O(W^2)` paths. Note, however, that not all these paths are relevant to the analysis of the behaviour of the ``Target.sum()`` method. When analyzing a piece of code we usually make the implicit assumption that its inputs must be *well-formed*. In the case of null-terminated linked lists, all the arrangements of list nodes that contain loops are ill-formed: If such "garbage" lists can be produced, this is due to a bug in the implementation of the ``LinkedList`` class, not of the ``sum`` method, and usually we are not interested in how this behaves when fed by garbage. In other words, while lazy initialization performs an exhaustive analysis of all the possible arrangements of the objects in the input heap, not all these arrangements are relevant to the analysis of the target code. Discarding them would meaningfully increment speed and precision of the analysis. In our example, JBSE should discard all the cases where the :math:`n_i` symbolic references are resolved by alias, retaining only the resolutions by null or by expansion. Were JBSE able to do that, the resulting symbolic execution tree would have the shape depicted in :num:`Figure #scanlist-symbolic-tree-wellformed`. If we bound the maximum number of ``LinkedList.Node`` objects to be at most :math:`W`, the total number of paths becomes :math:`1 + 1 + \ldots + 1 = 1 + \sum_{i = 1}^{W} 1 = 1 + W \in O(W)`, much less than without filtering out the irrelevant traces. Moreover, JBSE would discard all the diverging (infinite) paths except the one marked by a red dashed arrow in :num:`Figure #scanlist-symbolic-tree-wellformed`, that is unfeasible because the heap memory of a program always contains a finite, albeit unbounded, number of objects. This allows us to conclude that, when fed by well-formed linked lists, the ``Target.sum()`` method always converges and returns the sum of the values stored in the list.

.. _scanlist-symbolic-tree-wellformed:

.. figure:: /img/scanlist_symbolic_tree_wellformed.*
   :align: center
   :width: 700 px

   Symbolic execution tree for the list scanning program (only well-formed lists).

This example shows that excluding irrelevant inputs from the symbolic analysis of a piece of code is of paramount importance, both to make the analysis feasible within the typically limited computational resources available (time, memory), and to exclude spurious analysis results and unfeasible tests from its output. JBSE implements a number of techniques that empower its users by allowing them to specify rich classes of assumptions on the shape of the input heap objects. We will discuss them in details in the later chapters of this manual.

.. _Java Virtual Machine Specification (JVMS) books: https://docs.oracle.com/javase/specs/
.. _Hotspot: http://www.oracle.com/technetwork/java/javase/downloads/index.html
.. _OpenJ9: http://www.eclipse.org/openj9/
.. _JamaicaVM: https://www.aicas.com/wp/products-services/jamaicavm/
.. _Findbugs: http://findbugs.sourceforge.net/
.. _Checkstyle: http://checkstyle.sourceforge.net/
.. _PMD: http://pmd.sourceforge.net/
.. _as observed by Dijkstra: https://www.cs.utexas.edu/users/EWD/ewd02xx/EWD249.PDF
.. _Symbolic PathFinder: https://github.com/SymbolicPathFinder
.. _Sireum/Kiasan: http://www.sireum.org/
.. _JNuke: http://fmv.jku.at/jnuke/
.. _Wikipedia article: http://en.wikipedia.org/wiki/Symbolic_execution
.. _textbook: http://ix.cs.uoregon.edu/~michal/book/
.. _this paper: https://doi.org/10.1007/3-540-36577-X_40

.. [1] Actually, because of boxing and type erasure, the ``value`` field has type ``Object``, thus :math:`v_0` should be a symbolic reference that might potentially point to any ``Object``. To simplify the presentation we will suppose that it is an ``int`` symbolic value instead. Later in this book we will define techniques that allow to constrain ``list`` to have only elements with type ``Integer``.
