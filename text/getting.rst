#########################
Getting started with JBSE
#########################
In this section you will start to get your feet wet with JBSE. We will show its basic usage by analyzing a number of simple Java programs, including the ones presented in the Introduction. But first, let us discuss how you may obtain it.

*****************************
Obtaining and installing JBSE
*****************************
JBSE is an open-source project distributed according to the terms of the GNU General Public License version 3.0. Its source code is available at its Github repository https://github.com/pietrobraione/jbse. Right now there are not formal releases of JBSE, that must be installed by building it from source. Follow the instructions in the `README.md`_ file at the repository for instructions on how to build and deploy JBSE.

**********
Using JBSE
**********
JBSE is, first and foremost, a Java library. Compiling its source code will yield a jar file, to be linked against a main Java program that will use the functionalities it provides. No class in the JBSE jar file contains a ``public static void main(String[])`` method that you can invoke from the command line.

To start using JBSE we advise to install the latest Eclipse_ IDE, and import the JBSE project in an empty workspace by following the instructions in the `README.md`_ file at the JBSE repository. Then, create a new project with name ``example`` in the same Eclipse workspace where JBSE resides, and set its project dependencies to include JBSE. Add to the new project this class:

.. code-block:: java

   package smalldemos.ifx;

   import static jbse.meta.Analysis.ass3rt;

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
           ass3rt(a == b);
       }
   }

This is the "double-if" example presented in the Introduction, with the only difference that, instead of using the standard Java ``assert`` statement, we use a library method ``jbse.meta.Analysis.ass3rt`` that is more "friendly" to JBSE.

The easiest and most direct way to symbolically execute a Java method and obtain some feedback about the execution is to use the ``jbse.apps.run.Run`` class. This class demonstrates how one can build an application based on the JBSE library: Specifically, ``Run`` performs an exhaustive, possibly bounded symbolic execution of a prescribed Java method, by assigning symbolic values to all its parameters, including ``this`` if the method is not ``static``. During the symbolic execution, it prints to the console information about the progress of the execution. The ``Run`` application is highly configurable in the format and degree of detail of what it shows. On one extreme, it can be instructed to dump the full JVM state after the execution of each single bytecode; On the other, it is possible to make it emit just some end-of-execution statistics, and nothing else.

Coherently with what we said about the absence of a ``main`` method in the JBSE source code, the ``Run`` application is not invoked from the command line: We need to write a ``main`` method that creates a ``Run`` objects, configures it, and then asks it to start symbolic execution. Due to the high number of configuration parameters available for ``Run`` objects, configurations are encapsulated in suitable objects of class ``jbse.apps.run.RunParameters``. We therefore need to build a ``RunParameters`` object, use it to specify a set of symbolic execution and output parameters, and finally pass it as an argument to the constructor of a ``Run`` object. Finally, we start symbolic execution by invoking ``Run.run()``. Do this by creating this Java class in the ``example`` project:

.. code-block:: java

   package smalldemos.ifx;

   import jbse.apps.run.RunParameters;
   import jbse.apps.run.Run;
   ...

   public class RunIf {
       public static void main(String[] args)	{
           final RunParameters p = new RunParameters();
           set(p);
           final Run r = new Run(p);
           r.run();
       }
	
       private static void set(RunParameters p) {
           ...
       }
   }

Which parameters should we set, and how?

First, JBSE is a Java Virtual Machine. As with any Java Virtual Machine, be it symbolic or not, we must specify the classpath where JBSE will find the binaries of the program to be executed, the so-called *user classpath*. In this case the user classpath will contain two paths, one for the target ``smalldemos.ifx.IfExample`` class, and one for the ``jbse.meta.Analysis`` class that contains the ``ass3rt`` method invoked by ``m``. Note that under Eclipse all the binaries are emitted to a hidden ``bin`` project directory, and that the implicit execution directory of an Eclipse project is the project root directory. Given that the current directory will be the home of the ``example`` project, and supposing that the ``jbse`` git repository local clone is, say,  at ``/home/me/git/jbse``, the required paths should be approximately as follows:

.. code-block:: java

   ...
   public class RunIf {
       ...
       private static void set(RunParameters p) {
           p.addUserClasspath("./bin", "/home/me/git/jbse/target/classes");
           ...
       }
   } 

The ``RunParameters.addUserClasspath`` method is a varargs method, so you can list as many path strings as you want. Next, we must specify which method JBSE must run (remember, JBSE can symbolically execute *any* method). We do it by setting the method's *signature*:

.. code-block:: java

   ...
   public class RunIf {
       ...
       private static void set(RunParameters p) {
           p.addUserClasspath("./bin", "/home/me/git/jbse/target/classes");
           p.setMethodSignature("smalldemos/ifx/IfExample", "(I)V", "m");
           ...
       }
   } 

A method signature has three parts: The name in `internal classfile format`_ of the class that contains the method (``"smalldemos/ifx/IfExample"``), a `method descriptor`_ specifying the types of the method's parameters and of its return value (``"(I)V"``), and finally the name of the method (``"m"``). You can use the ``javap`` command, included with every JDK setup, to obtain the internal format signatures of methods: ``javap -s my.Class`` prints the list of all the methods in ``my.Class`` with their signatures in internal format.

Another essential parameter is the specification of which decision procedure JBSE must interface with in order to detect unfeasible paths. Without a decision procedure JBSE conservatively assumes that all paths are feasible. This is undesirable, since would allow to conclude, for instance, that every assertion you put in your code can be violated, be it possible or not. Supposing that you want to use Z3 and that the binary of Z3 is located at ``/opt/local/bin/z3``, you need to configure the ``RunParameters`` object as follows:

.. code-block:: java

   ...
   import static jbse.apps.run.RunParameters.DecisionProcedureType.Z3;

   public class RunIf {
       ...
       private static void set(RunParameters p) {
           p.addUserClasspath("./bin", "/home/me/git/jbse/target/classes");
           p.setMethodSignature("smalldemos/ifx/IfExample", "(I)V", "m");
           p.setDecisionProcedureType(Z3);
           p.setExternalDecisionProcedurePath("/opt/local/bin/z3");
           ...
       }
   } 

Now that we have set the parameters that allow the target code to be symbolically executed, we turn our attention to the parameters that customize the output. First, we ask JBSE to put a copy of the output in a dump file for offline inspection. At the purpose, create an ``out`` folder in the ``example`` project and add the following line to the ``set(RunParameters)`` method:

.. code-block:: java

   ...
   public class RunIf {
       ...
       private static void set(RunParameters p) {
           p.addUserClasspath("./bin", "/home/me/git/jbse/target/classes");
           p.setMethodSignature("smalldemos/ifx/IfExample", "(I)V", "m");
           p.setDecisionProcedureType(Z3);
           p.setExternalDecisionProcedurePath("/opt/local/bin/z3");
           p.setOutputFileName("./out/runIf_z3.txt");
           ...
       }
   }
 
Next, we specify which execution steps ``Run`` must show on the output. By default ``Run`` dumps the whole JVM symbolic state (path condition, stack, heap, static memory) after the execution of every bytecode, which is a bit exaggerated to our ends. We will therefore instruct the ``Run`` object not to print the unreachable objects and the standard library objects in the symbolic JVM states, and to omit some (scarecly interesting) path condition clauses. We will further reduce the amount of produced output by choosing to print only the *leaves* of the symbolic execution tree, i.e., the last states of all the execution traces.

.. code-block:: java

   ...
   import static jbse.apps.run.RunParameters.StateFormatMode.TEXT;
   import static jbse.apps.run.RunParameters.StepShowMode.LEAVES;

   public class RunIf {
       ...
       private static void set(RunParameters p) {
           p.addUserClasspath("./bin", "/home/me/git/jbse/target/classes");
           p.setMethodSignature("smalldemos/ifx/IfExample", "(I)V", "m");
           p.setDecisionProcedureType(Z3);
           p.setExternalDecisionProcedurePath("/opt/local/bin/z3");
           p.setOutputFileName("./out/runIf_z3.txt");
           p.setStateFormatMode(TEXT);
           p.setStepShowMode(LEAVES);
       }
   } 

Finally, run the ``RunIf`` class. The ``out/runIf_z3.txt`` file will contain something like this::

   This is the Java Bytecode Symbolic Executor's Run Tool (JBSE v.0.9.0-SNAPSHOT).
   Connecting to Z3 at /opt/local/bin/z3.
   Starting symbolic execution of method smalldemos/ifx/IfExample:(I)V:m at Sat Dec 15 10:06:40 CET 2018.
   .1.1[22] 
   Leaf state
   Path condition: 
           {R0} == Object[4727] (fresh) &&
           {V3} > 0 &&
           where:
           {R0} == {ROOT}:this &&
           {V3} == {ROOT}:x
   Heap: {
           Object[4727]: {
                   Origin: {ROOT}:this
                   Class: (2, smalldemos/ifx/IfExample)
                   Field[0]: Name: b, Type: Z, Value: true (type: Z)
                   Field[1]: Name: a, Type: Z, Value: true (type: Z)
           }
   }

   .1.1 trace is safe.
   .1.2[20] 
   Leaf state
   Path condition: 
           {R0} == Object[4727] (fresh) &&
           {V3} <= 0 &&
           where:
           {R0} == {ROOT}:this &&
           {V3} == {ROOT}:x
   Heap: {
           Object[4727]: {
                   Origin: {ROOT}:this
                   Class: smalldemos/ifx/IfExample
                   Field[0]: Name: b, Type: Z, Value: false (type: Z)
                   Field[1]: Name: a, Type: Z, Value: false (type: Z)
           }
   }

   .1.2 trace is safe.
   Symbolic execution finished at Sat Dec 15 10:06:43 CET 2018.
   Analyzed states: 729958, Analyzed traces: 2, Safe: 2, Unsafe: 0, Out of scope: 0, Violating assumptions: 0, Unmanageable: 0.
   Elapsed time: 2 sec 620 msec, Average speed: 278609 states/sec, Elapsed time in decision procedure: 7 msec (0,27% of total).

Let's analyze the output.

* ``{V0}``, ``{V1}``, ``{V2}``... (primitives) and ``{R0}``, ``{R1}``, ``{R2}``... (references) are the symbolic initial values of the program inputs. To track down which initial value a symbol correspond to (what we call the symbol's *origin*) you may read the ``Path condition:`` section of a final symbolic state. After the ``where:`` row you will find a sequence of equations that associate some of the symbols with their origins. The list is incomplete, but it contains the associations we care of. For instance you can see that ``{R0} == {ROOT}:this``; ``{ROOT}`` is a moniker for the *root frame*, i.e., the invocation frame of the initial method ``m``, and ``this`` indicates the "this" parameter. Overall, the equation means that the origin of ``{R0}`` is the instance of the ``IfExample`` class to which the ``m`` message is sent at the start of the symbolic execution. Similarly, ``{V3} == {ROOT}:x`` indicates that ``{V3}`` is the value of the ``x`` parameter of the initial ``m(x)`` invocation.
* ``.1.1[22]`` and ``.1.2[20]`` are the identifiers of the leaf symbolic states, i.e., the states that return from the initial ``m`` invocation to the (unknown) caller. The state identifiers follow the structure of the symbolic execution. The initial state has always identifier ``.1[0]``, and its immediate successors have identifiers ``.1[1]``, ``.1[2]``, etc. until JBSE must take some decision involving symbolic values. In this example, JBSE takes the first decision when it hits the first ``if (x > 0)`` statement. Since at that point of the execution ``x`` has still value ``{V3}`` and JBSE has not yet made any assumption on the possible value of ``{V3}``, two outcomes are possible: Either ``{V3} > 0``, and the execution takes the "then" branch, or ``{V3} <= 0``, and the execution takes the "else" branch. JBSE therefore produces *two* successor states, gives them the identifiers ``.1.1[0]`` and ``.1.2[0]``, and adds the assumptions ``{V3} > 0`` and ``{V3} <= 0`` to their respective path conditions. When the execution of the ``.1.1`` trace hits the second ``if`` statement, JBSE detects that the execution cannot take the "else" branch (otherwise, the path condition would be ``{V3} > 0 && {V3} <= 0 ...``, that has no solutions for any value of ``{V3}``) and does *not* create another branch. Similarly for the ``.1.2`` trace.
* The two leaf states can be used to extract *summaries* for ``m``. A summary is extracted from the path condition and the values of the variables and objects fields at a leaf state. In our example from the ``.1.1[22]`` leaf we can extrapolate that ``{V3} > 0 => {R0}.a == true && {R0}.b == true``, and from ``.1.2[20]`` that ``{V3} <= 0 => {R0}.a == false && {R0}.b == false``. This proves that for every possible value of the ``x`` parameter the execution of ``m`` always satisfies the assertion. 
* Beware! The dump shows the *final*, not the *initial* state of the symbolic execution. For example, while ``Object[0]`` is the initial ``this`` object, as stated by the path condition clause ``{R0} == Object[0]``, the values of its fields displayed at states ``.1.1[22]`` and ``.1.2[20]`` are the final, not the initial, ones. The initial, symbolic values of these fields are lost because the code under analysis never uses them. If you want to display all the details of the initial state, suitable step show modes exist.
* The last rows report some statistics. Here we are interested in the total number of traces (two traces, as discussed above), the number of *safe* traces, i.e., the traces that pass all the assertions (also two as expected), and the number of *unsafe* traces, that falsify some assertion (zero as expected). The dump also reports the total number of traces that violate an assumption (zero in this case, see later this section for a discussion of assumptions), and the total number of *unmanageable* traces. These are the traces that JBSE is not able to execute up to their leaves because of some limitation of JBSE itself.

***********
Assumptions
***********

An area where JBSE stands apart from all the other symbolic executors is its support to specifying custom *assumptions* on the symbolic inputs. Assumptions are indispensable to express preconditions over the input parameters of a method, invariants of data structures, and in general to constrain the range of the possible values of the symbolic inputs, either to exclude meaningless inputs, or just to reduce the scope of the analysis. Let us reconsider our running example and suppose that the method ``m`` has a precondition stating that it cannot be invoked with a value for ``x`` that is less than zero. Stating that a method has a precondition usually implies that we are not interested in analyzing how the method behaves when we pass to it parameters that violate its precondition. In other words, we want to *assume* that the inputs always satisfy the precondition, and analyze the behaviour of ``m`` under this assumption. The easiest way to introduce an assumption on the possible values of the ``x`` input is by injecting at the entry point of ``m`` a call to the ``jbse.meta.Analysis.assume`` method as follows:

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

When one needs to constrain symbolic *numeric* inputs, using ``jbse.meta.Analysis.assume`` can be enough. When one needs to enforce assumptions on symbolic *reference* inputs, using ``jbse.meta.Analysis.assume`` is in most cases unsuitable. This because ``jbse.meta.Analysis.assume`` evaluates its argument when it is invoked, which is OK for symbolic numeric inputs, but not in general for symbolic references since JBSE resolves a reference as soon as it is used (more precisely, as soon as it is loaded on the operand stack). Let us consider, for example, the linked list example of the Introduction and let's say we want to assume that the value stored in the fourth list item is different from ``0``. If we follow the previous pattern and inject at the method entry point the statement ``assume(list.header.next.next.next.value != 0)``, JBSE will first access ``{ROOT}:list``, then ``{ROOT}:list.header``, then ``{ROOT}:list.header.next``, then ``{ROOT}:list.header.next.next`` and then ``{ROOT}:list.header.next.next.next``. All these references are symbolic, and JBSE will resolve all of them, causing an early explosion of the total number of paths to be analyzed just to prune one of them. A possible way to avoid the issue is to manually move the ``assume`` right after the points where ``{ROOT}:list.header.next.next.next.value`` is accessed for the first time, a procedure that is in general complex and error-prone. It would be much better if the symbolic executor could automatically detect the first access, should it ever happen, and prune the violating trace on-the-fly. Another issue is that often we want to express assumptions over arbitrarily big sets of symbolic references. If, for example, we would like to assume that *all* the ``list`` items are nonzero, we should have a way to constrain *all* the symbolic values ``{ROOT}:list.header.value``, ``{ROOT}:list.header.next.value``, ``{ROOT}:list.header.next.next.value``... A similar problem arises if we want to specify the structural invariant stating that ``list`` shall have no loops. Expressing this kind of constraints by using ``Analysis.assume`` is impossible in many cases, and impractical in almost all the others.

JBSE allows to specify rich classes of assumptions on the shape of the input objects through a set of techniques that it implements.

* `Conservative repOk methods`_ are methods that validate the shape of a data structure by traversing it without resolving the unresolved symbolic references in it. JBSE will execute the conservative repOk methods on all the objects in the heap that have one, every time a symbolic reference is resolved. If the method detects that the resolution violates the structural invariant of the data structure where the reference is contained, then the trace is rejected.
* `LICS rules`_ use regular expressions to restrain the possible resolutions of (sets of) symbolic references. For instance, a rule ``{ROOT}:list.header(.next)* aliases nothing`` forbids all the symbolic references with origins ``{ROOT}:list.header``, ``{ROOT}:list.header.next``, ``{ROOT}:list.header.next.next``... to be resolved by alias, thus excluding the presence of loops.
* Triggers_ are user-defined instrumentation methods that JBSE executes right after the resolution of a symbolic reference matching a regular expression. Triggers have many uses: They can be used to update ghost variables, e.g., object counters as fresh objects are assumed by the expansion of symbolic references. They can also be used to automatically detect when a symbolic reference is first used, and e.g. perform a call of ``jbse.meta.Analysis.assume``, without having to manually detect the points in the code where the reference is first used.

.. _Eclipse: https://www.eclipse.org
.. _README.md: https://github.com/pietrobraione/jbse/blob/master/README.md
.. _internal classfile format: http://docs.oracle.com/javase/specs/jvms/se6/html/ClassFile.doc.html#14757
.. _method descriptor: http://docs.oracle.com/javase/specs/jvms/se6/html/ClassFile.doc.html#1169
.. _Conservative repOk methods: http://dx.doi.org/10.1145/1013886.1007526
.. _LICS rules: http://dx.doi.org/10.1145/2491411.2491433
.. _Triggers: http://dx.doi.org/10.1145/2491411.2491433
