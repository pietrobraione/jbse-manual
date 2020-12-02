##########
Using JBSE
##########

This section will describe in details how to use JBSE to symbolically execute Java bytecode programs and analyze them by means of assertions and assumptions.

*****************************
Performing symbolic execution
*****************************

The class that is responsible of performing symbolic execution of a program is the ``jbse.jvm.Engine`` class. It implements a symbolic JVM that can execute a Java method in a step-by-step fashion, i.e., one bytecode at time. In the case a state has more than one successor, the ``Engine`` selects one of them as the next state, and stores the others in an internal data structure. At the end of a path, the ``Engine`` can be backtracked to one of the previously stored states.

The ``jbse.jvm.Engine`` class offers a very low-level API, that allows a fine-grain control of symbolic execution, but is also challenging to use. For this reason, JBSE offers the ``jbse.jvm.Runner`` class, that drives an ``Engine`` object to perform an exhaustive symbolic execution of a method. A ``Runner`` object repeatedly steps and backtracks the ``Engine`` until it visits all (or some of)  the states in the symbolic execution tree of the method under execution. Users customize the behavior of a ``Runner`` object by registering callback methods that the ``Runner`` will invoke upon occurrence of prescribed events, e.g., before a bytecode step, at the entry of a method, upon backtrack, etc.

The ``jbse.apps.run.Run`` class is an example of an application that can be created by customizing the behavior of a ``Runner`` object. As discussed in the introduction, this class symbolically executes a method and emits information about the execution as, e.g., the traversed states, their path conditions, the shape of the symbolic execution tree, the assertion violations, etc.
