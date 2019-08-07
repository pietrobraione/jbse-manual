#########################
Getting started with JBSE
#########################
In this section you will start to get your feet wet with JBSE. We will show its basic usage by analyzing a number of simple Java programs, including the ones presented in the Introduction. But first, let us discuss how you may obtain it.

*****************************
Obtaining and installing JBSE
*****************************
JBSE is an open-source project distributed according to the terms of the GNU General Public License version 3.0. Its source code is available at its Github repository https://github.com/pietrobraione/jbse. Right now there are not formal releases of JBSE, that must be installed by building it from source. Follow the instructions in the `README.md`_ file at the repository for instructions on how to build and deploy JBSE.

***************
A basic example
***************
JBSE is, first and foremost, a Java library. Compiling the JBSE source code will yield a jar file that you need to link against a main Java program using it. No class in the JBSE jar file contains a ``public static void main(String[])`` method that you can invoke from the command line.

To start using JBSE we advise to install the latest Eclipse_ IDE, and import the JBSE project in an empty workspace by following the instructions in the `README.md`_ file at the JBSE repository. Then, create a new project with name ``example`` in the same Eclipse workspace where JBSE resides, and set its project dependencies to include the JBSE project. Add to the ``example`` project this class:

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

This is the "double-if" example presented in the Introduction, the only difference being that, instead of using the standard Java ``assert`` statement, we use a library method ``jbse.meta.Analysis.ass3rt`` that is more JBSE-friendly.

The easiest and most direct way to symbolically execute a Java method and obtain some feedback about the execution is to use the ``jbse.apps.run.Run`` class. This class demonstrates how one can build an application based on the JBSE library: Specifically, ``Run`` performs an exhaustive, possibly bounded symbolic execution of a prescribed Java method, by assigning symbolic values to all its parameters, including ``this`` if the method is not ``static``. During the symbolic execution, it prints to the console information about the progress of the execution. The ``Run`` application is highly configurable in the format and degree of detail of what it shows. On one extreme, it can be instructed to dump the full JVM state after the execution of each single bytecode. On the other, it is possible to make it emit just some end-of-execution statistics, and nothing else.

Coherently with what we said about the absence of a ``main`` method in the JBSE source code, the ``Run`` application cannot be invoked from the command line: We need to write a ``main`` method that creates a ``Run`` objects, configures it, and then asks it to start symbolic execution. Due to the high number of configuration parameters available for ``Run`` objects, configurations are encapsulated in suitable objects of class ``jbse.apps.run.RunParameters``. We therefore need to build a ``RunParameters`` object, use it to specify a set of symbolic execution and output parameters, and finally pass it as an argument to the constructor of a ``Run`` object. Finally, we start symbolic execution by invoking ``Run.run()``. Do this by creating this Java class in the ``example`` project:

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

First, we need to tell JBSE where is the program to execute. Being JBSE a Java Virtual Machine, we do this by specifying the classpath (more precisely, the so-called *user classpath*) where JBSE will look for the binaries (classfiles) of the program to be executed. The user classpath shall contain two paths, one pointing to the target ``smalldemos.ifx.IfExample`` class, and one pointing to the ``jbse.meta.Analysis`` class that contains the ``ass3rt`` method invoked by ``m``. Note that Eclipse emits the compiled ``smalldemos.ifx.IfExample`` class to a hidden ``bin`` directory in the ``example`` project. On converse, Eclipse builds the JBSE project by invoking Gradle, which puts the compiled classfiles of JBSE in a ``build/classes`` subdirectoy of the JBSE project, and emits the JAR files in a ``build/libs`` directory. Given that the implicit execution directory will be the home of the ``example`` project, and supposing that the ``jbse`` git repository local clone is, e.g.,  at ``/home/me/git/jbse``, the required paths should be approximately as follows:

.. code-block:: java

   ...
   public class RunIf {
       ...
       private static void set(RunParameters p) {
           p.addUserClasspath("./bin", "/home/me/git/jbse/build/classes");
           ...
       }
   } 

The ``RunParameters.addUserClasspath`` method is varargs, so you can list as many path strings as you want. Next, we must specify which method JBSE must run (remember, JBSE can symbolically execute *any* method). We do it by setting the method's *signature*:

.. code-block:: java

   ...
   public class RunIf {
       ...
       private static void set(RunParameters p) {
           p.addUserClasspath("./bin", "/home/me/git/jbse/build/classes");
           p.setMethodSignature("smalldemos/ifx/IfExample", "(I)V", "m");
           ...
       }
   } 

A method signature has three parts: The `binary name`_ of the class that contains the method (``"smalldemos/ifx/IfExample"``), a `method descriptor`_ specifying the types of the method's parameters and of its return value (``"(I)V"``), and finally the name of the method (``"m"``). You can use the ``javap`` command, included with every JDK setup, to obtain the internal format signatures of methods: ``javap -s my.Class`` prints the list of all the methods in ``my.Class`` with their signatures in internal format.

Another essential parameter is the specification of which decision procedure JBSE must interface with in order to detect unfeasible paths. Without a decision procedure JBSE conservatively assumes that all paths are feasible. This is undesirable, since it would allow to conclude, for instance, that every assertion you put in your code can be violated. Supposing that you want to use Z3 and that the Z3 binary is located, e.g., at ``/opt/local/bin/z3``, you need to configure the ``RunParameters`` object as follows:

.. code-block:: java

   ...
   import static jbse.apps.run.RunParameters.DecisionProcedureType.Z3;

   public class RunIf {
       ...
       private static void set(RunParameters p) {
           p.addUserClasspath("./bin", "/home/me/git/jbse/build/classes");
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
           p.addUserClasspath("./bin", "/home/me/git/jbse/build/classes");
           p.setMethodSignature("smalldemos/ifx/IfExample", "(I)V", "m");
           p.setDecisionProcedureType(Z3);
           p.setExternalDecisionProcedurePath("/opt/local/bin/z3");
           p.setOutputFileName("./out/runIf_z3.txt");
           ...
       }
   }
 
Next, we specify what of the symbolic execution ``Run`` shall display on the output. By default ``Run`` dumps the whole JVM symbolic state (path condition, stack, heap, static memory) after the execution of every single bytecode, which is a bit extreme, and slows down the execution considerably. We will therefore instruct the ``Run`` object to omit the unreachable objects and the standard library objects when printing a JVM symbolic state, and to omit some (scarecly interesting) path condition clauses. We will further reduce the amount of produced output by choosing to print only the *leaves* of the symbolic execution tree, i.e., the last states of all the execution traces.

.. code-block:: java

   ...
   import static jbse.apps.run.RunParameters.StateFormatMode.TEXT;
   import static jbse.apps.run.RunParameters.StepShowMode.LEAVES;

   public class RunIf {
       ...
       private static void set(RunParameters p) {
           p.addUserClasspath("./bin", "/home/me/git/jbse/build/classes");
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

.. _Eclipse: https://www.eclipse.org
.. _README.md: https://github.com/pietrobraione/jbse/blob/master/README.md
.. _binary name: https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.2.1
.. _method descriptor: https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.3.3
