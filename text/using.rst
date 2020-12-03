##########
Using JBSE
##########

This section will describe in details how to use JBSE to symbolically execute Java bytecode programs and analyze them by means of assertions and assumptions.

******************************
The symbolic execution classes
******************************

The class that is responsible of performing symbolic execution of a program is the ``jbse.jvm.Engine`` class. It implements a symbolic JVM that can execute an arbitrary Java method in a step-by-step fashion, i.e., one bytecode at time. In the case a state has more than one successor, the ``Engine`` selects one of them as the next state, and stores the others in an internal data structure. At the end of a path, the ``Engine`` can be backtracked to one of the previously stored states.

The ``jbse.jvm.Engine`` class offers a low-level API, that allows a fine-grain control of symbolic execution, but is also harder to use correctly. For this reason, JBSE offers the ``jbse.jvm.Runner`` class, that drives an ``Engine`` object to perform an exhaustive symbolic execution of a method. A ``Runner`` object repeatedly steps and backtracks the ``Engine`` until it visits all (or some of)  the states in the symbolic execution tree of the method under execution. Users customize the behavior of a ``Runner`` object by registering a listener object whose callback methods the ``Runner`` will invoke upon occurrence of prescribed events, e.g., before a bytecode step, at the entry of a method, upon backtrack, etc.

The ``jbse.apps.run.Run`` class is an example of an application that is created by customizing the behavior of a ``Runner`` object. As discussed in the introduction, this class symbolically executes a method and emits information about the execution as, e.g., the traversed states, their path conditions, the shape of the symbolic execution tree, the assertion violations, etc.

******************
Setting the engine
******************

The creation of an ``Engine`` is performed by a suitable Builder object of class ``jbse.jvm.EngineBuilder``. This object exposes a single method ``EngineBuilder.build(EngineParameters)``, that accepts as argument a ``jbse.jvm.EngineParameters`` object and returns a new ``Engine`` object. The ``EngineParameters`` object gathers the (many) configuration parameters that must be provided when the ``Engine`` is created. Similarly, a ``Runner`` is created by a ``jbse.jvm.RunnerBuilder`` object by means of its ``RunnerBuilder.build(RunnerParameters)`` method. The ``jbse.jvm.RunnerParameters`` class is very similar to the ``EngineParameter`` class, with the only difference that the former offers a superset of the methods of the latter to set the necessary additional parameters for a ``Runner`` to work. We now describe how an ``EngineParameters`` or a ``RunnerParameters`` object is set to suitably configure an ``Engine`` or a ``Runner``.

An ``Engine`` (and similarly a ``Runner``) must be provided the paths where it can find the classes of the application that it will symbolically execute and of the libraries the application will use, including the standard Java library. JBSE distinguishes, as Hotspot does, between the *bootstrap* classpath, where the standard library classes reside, the *extensions* classpath, home of the classes that must be loaded with the extensions mechanism, and the *user* classpath, where the classes belonging to the application to be run are put (the latter is what is usually meant when we talk about the "classpath" of an application). The bootstrap and the extensions classpath are, by default, automatically detected as the bootstrap and the extensions classpath of the JDK on which JBSE executes, but it is also possible to configure them manually. The bootstrap classpath can be set by specifying the home directory of a Java 8 JDK setup via the ``EngineParameters.setJavaHome(String)`` or ``EngineParameters.setJavaHome(Path)`` methods. To set the extensions classpath this must be first cleared from its default value by invoking ``EngineParameters.clearExtClasspath()``, then add paths to it by invoking ``EngineParameters.addExtClasspath(String...)`` or  ``EngineParameters.addExtClasspath(Path...)``. Similarly, to add a path to the user classpath either the  ``EngineParameters.addUserClasspath(String...)`` or the  ``EngineParameters.addExtClasspath(Path...)`` method can be used. The ``RunnerParameter`` class offers identical methods.

To indicate which method the ``Engine`` must symbolically execute, ``EngineParameter.setMethodSignature(String, String, String)`` must be invoked with the *signature* of the method as parameter. The signature of the method is given by the name of class it belongs to, its descriptor (which lists the types of its parameters and of its return value), and its name. The name of the class must be specified in `binary name`_ format: It must be fully qualified with the name of the package, and possibly of the other classes, that contain it, and the separator dots must be replaced by a slash (``/``) character in case the container on the left of the dot is a package, and by a dollar sign (``$``) in case the container is a class. For example the class ``java.util.LinkedList`` has binary name ``java/util/LinkedList``, and its nested class ``java.util.LinkedList.Node`` has binary name ``java/util/LinkedList$Node``. The `descriptor`_ is necessary to disambiguate overloaded methods, having same name and different parameters lists. It is composed by a list of parameters types, enclosed in parentheses, followed by the type of the return value or ``V``, if the method is declared ``void``, The types of the parameters must be encoded as type descriptors according to `Table 4.3-A`_ of the Java Virtual Machine Specification. We report a shortened version of the table for reference:

=================   =====================
Java type           Type descriptor
=================   =====================
``byte``            ``B``
``char``            ``C``
``double``          ``D``
``float``           ``F``
``int``             ``I``
``long``            ``J``
*Class*             ``L`` *Class* ``;``
*JavaType* ``[]``   ``[`` *TypeDescriptor*
=================   =====================

For instance, if we want to symbolically execute a method ``double m(int, Object[][])`` of class ``foo.Baz``, we must set the ``EngineParameters`` by invoking ``setMethodSignature("foo/Baz" "(I[[Ljava/lang/Object;)D", "m")``. 


.. _binary name: https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.2.1
.. _descriptor: https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.3.3
.. _Table 4.3-A: https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.3.2-200


