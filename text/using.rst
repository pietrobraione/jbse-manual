##########
Using JBSE
##########

This section will describe in details how to use JBSE to symbolically execute Java bytecode programs and analyze them by means of assertions and assumptions.

*****************************
Performing symbolic execution
*****************************

The class that is responsible of performing symbolic execution of a program is the ``jbse.jvm.Engine`` class. It implements a symbolic JVM that can execute a Java method in a step-by-step fashion, i.e., one bytecode at time. In the case a state has more than one successor, the ``Engine`` selects one of them as the next state, and stores the others in an internal data structure. At the end of a path, the ``Engine`` can be backtracked to one of the previously stored states.

The ``jbse.jvm.Engine`` class offers a very low-level API, that allows a fine-grain control of symbolic execution, but is also challenging to use. For this reason, JBSE offers the ``jbse.jvm.Runner`` class, that drives an ``Engine`` object to perform an exhaustive symbolic execution of a method. A ``Runner`` object repeatedly steps and backtracks the ``Engine`` until it visits all (or a part of)  the states in the symbolic execution tree of the method under execution. Users customize the behavior of a ``Runner`` object by register callback methods that the ``Runner`` will invoke upon occurrence of prescribed events (e.g., before a bytecode step, at the entry of a method, upon backtrack, etc.).

The ``jbse.apps.run.Run`` class is an example of an application that can be created by customizing the behavior of a ``Runner`` object. As discussed in the introduction, this class symbolically executes a method and emits information about the execution as, e.g., the traversed states, their path conditions, the shape of the symbolic execution tree, the assertion violations, etc.

**************************
Assertions and assumptions
**************************

An area where JBSE stands apart from all the other symbolic executors is its support to specifying custom *assumptions* on the symbolic inputs. Assumptions are indispensable to express preconditions over the input parameters of a method, invariants of data structures, and in general to constrain the range of the possible values of the symbolic inputs, either to exclude meaningless values, or just to reduce the scope of the analysis. Let us reconsider our running example and suppose that the method ``m`` has a precondition stating that it cannot be invoked with a value for ``x`` that is less than zero. Stating that a method has a precondition usually implies that we are not interested in analyzing how the method behaves when its inputs violate the precondition. In other words, we want to *assume* that the inputs always satisfy the precondition, and analyze the behaviour of ``m`` under this assumption. The easiest way to introduce an assumption on the possible values of the ``x`` input is by injecting at the entry point of ``m`` a call to the ``jbse.meta.Analysis.assume`` method as follows:

.. code-block:: java

   ...
   import static jbse.meta.Analysis.assume;

   public class IfExample {
       boolean a, b;
       public void m(int x) {
           assume(x > 0);
           if (x > 0) {
           ...
       }
   }

When JBSE hits a ``jbse.meta.Analysis.assume`` method invocation it evaluates its argument, then it either continues the execution of the trace (if ``true``) or discards it and backtracks to the next trace (if ``false``). With the above changes the last rows of the dump will be as follows::

   ...
   .1.2 trace violates an assumption.
   Symbolic execution finished at Sat Dec 15 10:26:55 CET 2018.
   Analyzed states: 729950, Analyzed traces: 2, Safe: 1, Unsafe: 0, Out of scope: 0, Violating assumptions: 1, Unmanageable: 0.
   Elapsed time: 2 sec 625 msec, Average speed: 278076 states/sec, Elapsed time in decision procedure: 7 msec (0,27% of total).

The total number of traces is still two, but now JBSE reports that one of the traces violates an assumption. Putting the ``assume`` invocation at the entry of ``m`` ensures that the useless traces are discarded as soon as possible.

When one needs to constrain symbolic *numeric* inputs, using ``jbse.meta.Analysis.assume`` can be enough. When one needs to enforce assumptions on symbolic *reference* inputs, using ``jbse.meta.Analysis.assume`` is in most cases unsuitable. This because ``jbse.meta.Analysis.assume`` evaluates its argument when it is invoked, which is OK for symbolic numeric inputs, but not in general for symbolic references since JBSE resolves a reference as soon as it is used (more precisely, as soon as it is loaded on the operand stack). Let us consider, for example, the linked list example of the Introduction and let's say we want to assume that the value stored in the fourth list item is different from ``0``. If we follow the previous pattern and inject at the method entry point the statement ``assume(list.header.next.next.next.value != 0)``, JBSE will first access ``{ROOT}:list``, then ``{ROOT}:list.header``, then ``{ROOT}:list.header.next``, then ``{ROOT}:list.header.next.next`` and then ``{ROOT}:list.header.next.next.next``. All these references are symbolic, and JBSE will resolve all of them, causing an early explosion of the total number of paths to be analyzed just to prune one of them. A possible way to avoid the issue is to manually move the ``assume`` right after the points where ``{ROOT}:list.header.next.next.next.value`` is accessed for the first time, a procedure that is in general complex and error-prone. It would be much better if the symbolic executor could automatically detect the first access, should it ever happen, and prune the violating trace on-the-fly. Another issue is that often we want to express assumptions over arbitrarily big sets of symbolic references. If, for example, we would like to assume that *all* the ``list`` items are nonzero, we should have a way to constrain *all* the symbolic values ``{ROOT}:list.header.value``, ``{ROOT}:list.header.next.value``, ``{ROOT}:list.header.next.next.value``... A similar problem arises if we want to specify the structural invariant stating that ``list`` shall have no loops. Expressing this kind of constraints by using ``Analysis.assume`` is impossible in many cases, and impractical in almost all the others. For this reason JBSE allows to specify rich classes of assumptions on the shape of the input objects by means of a number of techniques: `Conservative repOk methods`_, `LICS rules`_, and triggers_.

* Conservative repOk methods validate the shape of a data structure by traversing it without resolving the unresolved symbolic references in it. Every time JBSE resolves a symbolic reference, it executes the conservative repOk method on all the objects that have one. If the execution of a conservative repOk on an object detects that the object violates its structural invariant, the trace is rejected.
* LICS rules restrain the possible resolutions of (sets of) symbolic references specified by means of regular expressions. For instance, a rule ``{ROOT}:list/header(/next)* aliases nothing`` forbids resolution by alias of all the symbolic references with origins ``{ROOT}:list.header``, ``{ROOT}:list.header.next``, ``{ROOT}:list.header.next.next``... thus excluding the presence of loops between list nodes.
* Triggers are user-defined instrumentation methods that JBSE executes right after the resolution of a symbolic reference matching a prescribed regular expression. Triggers can be used to update ghost variables, e.g., to update an object counter as a fresh objects is assumed by the expansion of symbolic references. They can also be used to automatically detect when a symbolic reference is first used and, e.g., invoke ``jbse.meta.Analysis.assume`` without having to manually detect the points in the code where the reference is first used.

.. _Conservative repOk methods: https://doi.org/10.1145/1013886.1007526
.. _LICS rules: https://doi.org/10.1145/2491411.2491433
.. _triggers: https://doi.org/10.1145/2491411.2491433
