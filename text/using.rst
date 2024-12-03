##########
Using JBSE
##########

This section will describe in details how to use JBSE to symbolically execute Java bytecode programs and analyze them by means of assertions and assumptions.

******************************
The symbolic execution classes
******************************

The class that is responsible of performing symbolic execution of a program is ``jbse.jvm.Engine``. It implements a symbolic JVM that can execute an arbitrary Java method in a step-by-step fashion, i.e., one bytecode at time. In the case a symbolic state has more than one successor, the ``Engine`` selects one of them as the next state, and stores the others in an internal data structure. At the end of a path, the ``Engine`` can be backtracked to one of the previously stored states.

The ``Engine`` class offers a low-level API, that allows a fine-grain control of symbolic execution, but is also quite tedious to use correctly. For this reason, JBSE offers the ``jbse.jvm.Runner`` class. A ``Runner`` object drives an ``Engine`` to perform an exhaustive symbolic execution of a method. To this end, the ``Runner`` repeatedly steps and backtracks the ``Engine`` until it visits all (or a suitable subset of) the states in the symbolic execution tree of the method under execution. Users can customize the behavior of a ``Runner`` object by registering a suitable listener object that provides a set of callback methods. The ``Runner`` will invoke the appropriate callback method upon occurrence of an event, e.g., before or after a bytecode step, at the entry of a method, upon backtrack, etc.

By injecting behavior through callbacks it is possible to customize the behavior of a ``Runner`` object and create applications. The ``jbse.apps.run.Run`` class is an example of an application that is done this way. As discussed in the introduction, this application symbolically executes a method and prints to the console information about the execution as, e.g., the traversed states, their path conditions, the shape of the symbolic execution tree, the violations of the assertions, etc. We will introduce the capabilities of the ``Run`` application in a later section, while this section focuses on the use of the ``Engine`` and ``Runner`` classes.

****************************
Creating a symbolic executor
****************************

The creation of an ``Engine`` is performed by a suitable Builder object of class ``jbse.jvm.EngineBuilder``. This object exposes a single method ``EngineBuilder.build(EngineParameters)``, that accepts as argument an object of class ``jbse.jvm.EngineParameters`` and returns a new ``Engine``. The ``EngineParameters`` object gathers the (many) configuration parameters that must be provided to create an ``Engine``. Similarly, a ``Runner`` must be created via a ``jbse.jvm.RunnerBuilder`` object by invoking its ``RunnerBuilder.build(RunnerParameters)`` method. The ``jbse.jvm.RunnerParameters`` class offers a superset of the methods of the ``EngineParameter`` class, i.e., all the methods to configure the parameters necessary to create the ``Engine`` underlying the ``Runner``, plus the methods to configure the parameters specific to the ``Runner`` (e.g., the listener). We now describe how an ``EngineParameters`` or a ``RunnerParameters`` object is set to suitably configure an ``Engine`` or a ``Runner``.

=================
Setting the paths
=================

An ``Engine`` (and similarly a ``Runner``) must know the paths where it can find the classes of the application that it must symbolically execute, and of the libraries the application invokes, including the standard Java library. Mirroring what Oracle's as Hotspot JVM does, JBSE distinguishes the *bootstrap* classpath, where the core JDK classes reside, the *extensions* classpath, home of the classes that must be loaded with the extensions mechanism, and the *user* classpath, pointing to the classes that belong to the application to be run (the latter is what we usually mean when we talk about the "classpath of an application"). The bootstrap and the extensions classpath are, by default, automatically configured by the ``EngineParameters`` and ``RunnerParameters`` objects by detecting the bootstrap and the extensions classpath of the JDK on which JBSE executes. This means that in most cases you do not need to configure them manually. Should you ever need to override this default, you can set the bootstrap classpath by specifying the home directory of a Java 8 JDK setup via the ``EngineParameters.setJavaHome(String)`` or ``EngineParameters.setJavaHome(Path)`` methods. To modify the extensions classpath you may add paths to it by invoking ``EngineParameters.addExtClasspath(String...)`` or  ``EngineParameters.addExtClasspath(Path...)``. If you want to completely override it, you need to first clear it from its default value by invoking ``EngineParameters.clearExtClasspath()``. Finally, to add paths to the user classpath use either the  ``EngineParameters.addUserClasspath(String...)`` or the  ``EngineParameters.addExtClasspath(Path...)`` method (by default, the user classpath is empty). The ``RunnerParameter`` class offers identical methods.

=========================
Setting the target method
=========================

To specify which method the ``Engine`` must symbolically execute, invoke ``EngineParameter.setMethodSignature(String, String, String)`` with the *signature* of the method. The signature of the method is composed by three strings: The name of class the method belongs to, the method descriptor (a string listing the types of the method parameters and of the return value), and the method's name. The name of the class must be specified in `binary name`_ format: It must be fully qualified with the name of the package, and possibly of the other classes, that contain it, the separator dots must be replaced by slash (``/``) characters in case the container on the left of the dot is a package, and by a dollar sign (``$``) in case the container is a class. For example the binary name of the class ``java.util.LinkedList`` is ``java/util/LinkedList``, and the binary name of the nested class ``java.util.LinkedList.Node`` is ``java/util/LinkedList$Node``. The `descriptor`_ is necessary to distinguish overloaded methods. It is composed by a list of parameters types, enclosed in parentheses, followed by the type of the return value, or ``V`` if the method is declared ``void``. The types of the parameters must be encoded as type descriptors according to `Table 4.3-A`_ of the Java Virtual Machine Specification. 

.. _type-descriptors:

.. table:: Type descriptors for Java types
   :align: center
   :width: 300 px
   :widths: auto

   ==================   =============================
   Java type            Type descriptor
   ==================   =============================
   ``byte``             ``B``
   ``char``             ``C``
   ``double``           ``D``
   ``float``            ``F``
   ``int``              ``I``
   ``long``             ``J``
   *ClassName*          ``L`` *ClassBinaryName* ``;``
   *JavaType*\ ``[]``   ``[``\ *TypeDescriptor*
   ==================   =============================

:numref:`type-descriptors` reports the correspondence between Java types and types descriptors for reference. As an example, if we want to symbolically execute a method ``double m(int, Object[][])`` of class ``foo.Baz``, we must configure the ``EngineParameters`` object by invoking ``setMethodSignature("foo/Baz", "(I[[Ljava/lang/Object;)D", "m")``.

======================
Setting the calculator
======================

An ``Engine`` needs to be able to create, from time to time, new symbolic expression. When this happens, some basic manipulations are usually performed on the created symbolic expression at the purpose of simplifying it. For example, it is possible to configure an ``Engine`` so it, whenever it must add the symbol ``a`` to the number ``0``, simplifies the resulting expression ``a + 0`` to ``a``. To this end, an ``Engine`` depends on an object extending the abstract class ``jbse.val.Calculator``, that it uses to create and manipulate all the symbolic expressions, and that performs all the simplifications on-the-fly. It is therefore necessary to inject the dependency to a suitable subclass of ``Calculator`` through the method ``setCalculator(Calculator)`` of the classes ``EngineParameters`` and ``RunnerParameters``. Currently, the only concrete subclass of ``Calculator`` is the class ``jbse.rewr.CalculatorRewriting``. A ``CalculatorRewriting`` applies a set of rewriting rules to simplify all the symbolic expressions it produces. It is possible to plug the rewriting rules, implemented as subclasses of ``jbse.rewr.RewriterCalculatorRewriting``, by invoking the ``CalculatorRewriting.addRewriter(RewriterCalculatorRewriting)`` method. The package ``jbse.rewr`` contains a collection of rewriting rules performing some useful simplifications. The most important ones, that are de facto compulsory, are:

* ``jbse.rewr.RewriterExpressionOrConversionOnSimplex``: necessary to simplify all the expressions whose operands are numeric, e.g., to simplify ``3 + 2`` to ``5``;
* ``jbse.rewr.RewriterFunctionApplicationOnSimplex``: similar to the previous, where the operator is a (symbolic) function application as ``sin``, ``cos``, ``max``, ``min``...
* ``jbse.rewr.RewriterZeroUnit``: simplifies some operations with zero or one that have trivial result: e.g., simplifies ``a * 0`` to ``0``, and ``1 * b`` to ``b``;
* ``jbse.rewr.RewriterNegationElimination``: eliminates double negations simplifying, e.g., ``- (- a)`` to ``a``.

The other rewriters in the package ``jbse.rewr`` can be used to simplify nonlinear expression with trigonometric operators and square roots. Historically they have been used to check properties involving distances in the Cartesian plane and polar-to-cartesian and their inverse coordinates conversions. A more mundane setup of JBSE would be as follows:

.. code-block:: java

   import jbse.jvm.EngineParameters;
   import jbse.rewr.CalculatorRewriting;
   import jbse.rewr.RewriterExpressionOrConversionOnSimplex;
   import jbse.rewr.RewriterFunctionApplicationOnSimplex;
   import jbse.rewr.RewriterNegationElimination;
   import jbse.rewr.RewriterZeroUnit;
   ...
   
   EngineParameters p = new EngineParameters();
   ...
   CalculatorRewriting calc = new CalculatorRewriting();
   calc.addRewriter(new RewriterExpressionOrConversionOnSimplex());
   calc.addRewriter(new RewriterFunctionApplicationOnSimplex());
   calc.addRewriter(new RewriterZeroUnit());
   calc.addRewriter(new RewriterNegationElimination());
   p.setCalculator(calc);


Unfortunately the order the rewriters are added to the calculator matters. Moreover, some rewriters depend on the presence of other rewriters. Refer the Javadoc of the rewriters classes for more information.


===============================
Setting the decision procedures
===============================

To  prune the unfeasible branches of the symbolic execution tree an ``Engine`` must decide whether a symbolic expression of boolean type is satisfiable. To this end, the ``Engine`` uses an object with class ``jbse.dec.DecisionProcedureAlgorithms``. Upon configuration it is necessary to create this object and inject the dependency by invoking the ``setDecisionProcedure(DecisionProcedureAlgorithms)`` method exposed by the ``EngineParameters`` and ``RunnerParameters`` classes. This because a ``DecisionProcedureAlgorithms`` object can be configured according to the capabilities it needs to provide.

For the sake of simplicity, a JBSE decision procedure object recognizes a proper subset of the possible boolean clauses produced by an ``Engine``. As a consequence, no single decision procedure object is able to decide the satisfiability of all the clauses generated by all the possible symbolic executions. For this reason, decision procedure objects must be organized in a Chain of Responsibility: Whenever a decision procedure is unable to decide the satisfiability of a clause, it delegates the task to the next decision procedure in the chain.

A JBSE decision procedure class extends the abstract class ``jbse.dec.DecisionProcedure``, and provides a set of methods to check the satisfiability of all the possible boolean clauses that JBSE may produce. The class ``DecisionProcedureAlgorithms``, on the other hand, is a Decorator adding to an arbitrary ``DecisionProcedure`` a set of methods that, for each bytecode instruction, perform the correct sequences of satisfiability queries necessary to implement the bytecode semantics. The necessary steps to configure the ``Engine`` are therefore:

* Creating a set of ``DecisionProcedure`` objects to decide a sufficient subset of the clauses that might appear in the path conditions of the program under analysis, and arranging them in a Chain of Responsibility;
* Wrapping the topmost ``DecisionProcedure`` in the Chain of Responsibility in a ``DecisionProcedureAlgorithms`` object;
* Setting the ``EngineParameters`` or ``RunnerParameters`` object via the ``setDecisionProcedure(DecisionProcedureAlgorithms)`` method.

Most decision procedures are declared in the package ``jbse.dec``. They are typically configured through their constructor, that is also used to set the next decision procedure in the Chain of Responsibility. The most important classes are:

* ``jbse.dec.DecisionProcedureClassInit``: This decision procedure recognizes only the clauses that predicate on the initialization status of a class (i.e., whether a class must be assumed to be loaded before the start of symbolic execution or not). It is indispensable, and therefore shall always be present in a Chain of Responsibility of decision procedures.
* ``jbse.dec.DecisionProcedureSMTLIB2_AUFNIRA``: This decision procedure interacts via console with any SMT solver that is compliant with the SMTLIB 2 standard and that supports the AUFNIRA logic. Currently the SMT solver is used to decide only the numeric clauses. The only SMT solvers we are sure that work with JBSE are Z3 and CVC4.
* ``jbse.dec.DecisionProcedureLICS``: This decision procedure implements the LICS rule language that allows to restrain the possible resolutions of (sets of) symbolic references. It therefore recognizes the clauses that predicate on symbolic references.
* ``jbse.dec.DecisionProcedureAlwSat``: This decision procedure is a dummy decision procedure that recognizes all the clauses and always answers that a clause is satisfiable. It must be used as the last decision procedure in the Chain of Responsibility.

These classes typically yield, when combined, a sufficiently powerful and flexible solver. A possible configuration example code follows:

.. code-block:: java

   import jbse.dec.DecisionProcedureAlwSat;
   import jbse.dec.DecisionProcedureClassInit;
   import jbse.dec.DecisionProcedureLICS;
   import jbse.dec.DecisionProcedureSMTLIB2_AUFNIRA;
   import jbse.jvm.EngineParameters;
   import jbse.rewr.CalculatorRewriting;
   import jbse.rules.ClassInitRulesRepo;
   import jbse.rules.LICSRulesRepo;

   EngineParameters p = new EngineParameters();
   ...
   CalculatorRewriting calc = new CalculatorRewriting();
   ...
   p.setCalculator(calc)
   
   ArrayList<String> z3CommandLine = new ArrayList<>();
   z3CommandLine.add("/opt/local/bin/z3");
   z3CommandLine.add("-smt2");
   z3CommandLine.add("-in");
   z3CommandLine.add("-t:100");
   LICSRulesRepo licsRules = new LICSRulesRepo();
   ClassInitRulesRepo initRules = new ClassInitRulesRepo();
   ...
   DecisionProcedureAlwSat decAlwSat = new DecisionProcedureAlwSat(calc);
   DecisionProcedureSMTLIB2_AUFNIRA decSMT = new DecisionProcedureSMTLIB2_AUFNIRA(decAlwSat, z3CommandLine);
   DecisionProcedureLICS decLICS = new DecisionProcedureLICS(decSMT, licsRules) 
   DecisionProcedureClassInit decInit = new DecisionProcedureClassInit(decLICS, initRules); 
   DecisionProcedureAlgorithms decAlgo = new DecisionProcedureAlgorithms(decInit);
   p.setDecisionProcedure(decAlgo);

We add some final remarks:

* The ``DecisionProcedureAlwSat`` constructor accepts as parameter a ``Calculator``. It is good practice (although not strictly necessary) to pass the same calculator object passed to the ``EngineParameters`` or ``RunnerParameters`` object via the  ``setCalculator`` method.
* A ``DecisionProcedureSMTLIB2_AUFNIRA`` must receive as a constructor parameter the command line that must be used to invoke the SMT solver. This is of course platform- and environment-dependent. In the above example we proposed a possible command line for invoking Z3 in a UNIX-like environment.
* Differently from other decision procedures, where it is possible to interactively send and retract assertions, the ``DecisionProcedureLICS`` and ``DecisionProcedureClassInit`` objects must be configured at construction time by passing suitable objects (with class ``jbse.rules.LICSRulesRepo`` and ``jbse.rules.ClassInitRulesRepo``, respectively) that gather the assertions about reference resolution and class initialization, respectively. We will discuss these constraints in a later section.

===================
Building the engine
===================

Once set an ``EngineParameters`` object, we are ready to create an ``Engine``; It is sufficient to create an ``EngineBuilder`` object and invoke its ``build`` method as follows:

.. code-block:: java

   ...
   import jbse.jvm.EngineParameters;
   ...
   
   EngineParameters p = new EngineParameters();
   ...
   EngineBuilder b = new EngineBuilder();
   Engine e = b.build(p);

====================
Setting the listener
====================

The procedure for creating a ``Runner`` is mostly identical to the one for creating an ``Engine``, with the obvious difference that the involved classes are ``RunnerParameters`` and ``RunnerBuilder`` (the name of the methods are identical). The true distinguishing feature is that a ``RunnerParameters`` object allows to register a listener for the symbolic execution process: Differently from an ``Engine``, that must be stepped bytecode-by-bytecode, a ``Runner`` fully executes a method symbolically without interruption. To allow a degree of observability and controllability of a ``Runner``'s symbolic execution, this notifies a listener object extending the ``jbse.jvm.Runner.Actions`` class at appropriate moments---before a bytecode step, after a bytecode step, at the entry of a method... An application may thus monitor an execution by defining a suitable listener object and registering it by invoking the ``setActions(Runner.Actions)`` method of the ``RunnerParameters`` object:

.. code-block:: java
 
   ...
   import jbse.jvm.Runner;
   import jbse.jvm.Runner.Actions;
   import jbse.jvm.RunnerBuilder;
   import jbse.jvm.RunnerParameters;
   ...
   
   RunnerParameters p = new RunnerParameters();
   ...
   Actions a = new Runner.Actions() {
     @Override
     public boolean atStart() {
       ...
     }

     @Override
     public boolean atStepPre() {
       ...
     }

     ...
   }
   p.setActions(a);

   RunnerBuilder b = new RunnerBuilder();
   Runner r = b.build(p);

For maximum observability and controllability of the symbolic execution process it is possible at any time to extract the ``Engine`` object underlying a ``Runner``.

***************************
Using the symbolic executor
***************************

The symbolic execution process performed by JBSE is simple and complex at the same time. On one hand the gist of the process is easy to grasp:

* Execute the next bytecode;
* If you are at a branch in the symbolic execution tree select a way to go (otherwise, follow the only way to go);
* Repeat.

When there are no more bytecodes to execute (i.e., the state of symbolic execution is *stuck*) it is possible to *backtrack* to one of the previously visited branching points in the symbolic execution tree and follow an unvisited direction, until all the paths of the symbolic execution tree are covered, or some budget is exhausted, whatever comes first.

On the other hand, JBSE aims at being as conformant as possible to the Java Virtual Machine Specification v.8, and as behaviorally similar as possible to its Hotspot implementation. This means, for instance, that JBSE follows quite closely the Hotspot bootstrap process and loads and initializes the same large set of core classes, in the same order as Hotspot. The consequence of such a degree of compliance is that JBSE must manage complex and large application states, even before it starts executing the user's code. Symbolic execution of a method is performed in phases: the *pre-initial* phase bootstraps the execution by loading and initializing the core classes, then an initial state is created where the frame of the target method is pushed on the stack of the state emerging from the pre-initial phase, and finally during the *post-initial* phase the target method is actually executed.

================
Using the engine
================

The ``Engine`` class exposes a low-level interface to symbolic execution. After its creation, an ``Engine`` must be initialized by invoking its ``init()`` method, and after that it can be stepped by repeatedly invoking its ``step()`` method as long as a successor state exists, a thing that can be checked by invoking the ``canStep()`` method. This happens until the execution state gets stuck, or you decide to prematurely end the exploration of the current execution path by invoking the ``stopCurrentPath()`` method. 

.. code-block:: java

   import jbse.jvm.Engine;

   Engine e;
   ...
   e.init();
   while (e.canStep()) {
     e.step();

     ...
     //possibly:
     e.stopCurrentPath();
     ...
   }
   
When the ``Engine`` is at a branch point in the symbolic execution tree, it automatically selects one of the successor states as the next one. The ``step()`` method returns a Memento ``jbse.tree.StateTree.BranchPoint`` object, that can be used to detect whether the ``Engine`` is passing through a same branch point again. To backtrack to the next pending branch in the symbolic execution tree invoke the ``backtrack()`` method, not before checking whether such a pending branch exists with the method ``canBacktrack()``.

.. code-block:: java

   import jbse.jvm.Engine;

   Engine e;
   ...
   e.init();
   while (true) {
     while (e.canStep()) {
       e.step();
       ...
     }
     if (e.canBacktrack()) {
       e.backtrack();
     } else {
       break;
     }
   }

By following this pattern of stepping-plus-backtracking JBSE will perform a depth-first visit of the symbolic execution tree.

================
Using the runner
================

The ``Runner`` class offers a higher-level experience to running JBSE. Once configured, it is sufficient to invoke the ``run()`` method, and the ``Runner`` will automatically perform a step-backtrack loop similar to the one presented earlier. By itself this loop will silently traverse the symbolic execution tree and (possibly) terminate: To introduce some interesting behavior it is necessary to implement some ``Actions`` listener object and configure the ``Runner`` with it. At a minimum, the listener can monitor the symbolic execution and report its progress in some way. The ``Runner`` object, however, also offers two ways to control the extent of the execution. First, every method of the ``Actions`` class returns a boolean value, either true, in case you want to early stop the execution, or false, in case you want to continue it. A finer control on the explored region of the state space can be achieved by configuring the *scope* parameters of the ``RunnerParameters`` object. The scope parameters allow you to define bounds above which the exploration of a trace is abandoned. The *heap scope* is the maximum number of objects of a given class that may be present in a state's heap; The *depth scope* is the maximum number of backtrack point that may be present from the start of the execution along a trace; And finally, the *count scope* is the maximum number of steps that may be performed after a backtrack point and before the next one. For example, if the depth scope is 3 we may have a trace up to depth .1.1.1, but not to depth .1.1.1.1, and if the count scope is 100 we may have a trace up to .1[100] but not to ,1[101]. An example of use of a runner is the following:

.. code-block:: java
 
   ...
   import jbse.jvm.Runner;
   import jbse.jvm.Runner.Actions;
   import jbse.jvm.RunnerBuilder;
   import jbse.jvm.RunnerParameters;
   ...
   
   RunnerParameters p = new RunnerParameters();
   ...
   Actions a = new Runner.Actions() { ... }
   p.setActions(a);

   p.setHeapScope(2, "java/util/LinkedList");
   p.setHeapScope(10, "java/util/LinkedList$Node");
   p.setDepthScope(100);
   p.setCountScope(10000);

   RunnerBuilder b = new RunnerBuilder();
   Runner r = b.build(p);

   r.run();

.. _binary name: https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.2.1
.. _descriptor: https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.3.3
.. _Table 4.3-A: https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.3.2-200


