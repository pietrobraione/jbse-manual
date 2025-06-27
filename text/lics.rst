#######################
LICS rules and triggers
#######################

This section will describe in details how to specify assumptions on the shape of the initial heap (object) memory by means of LICS rules and triggers.

***********************
An illustrative example
***********************
Let us reconsider the ``LinkedList`` example proposed in :ref:`ssec-introduction-symbolic`. We report the main definitions for ease of reading: 

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

   public class LinkedList<I> {
       private Node head;

       private static class Node {
           private I value;
           private Node next;
           ...
       }
       ...
   }

Let us say that we want to symbolically execute the ``Target.sum(List)`` method. Two questions arise:

* How do we instruct JBSE to consider ``LinkedList`` as a possible compatible type for the expansion of the ``{ROOT}:list`` symbolic reference? [1]_
* How do we specify that we are not interested in analyzing the ill-formed input lists reported in :numref:`scanlist-symbolic-tree`?

The second problem is tantamount to assuming that the shape of the initial object subgraph starting from ``{ROOT}:list`` is that of a well-formed list, or in other words, that such subgraph satisfies the *representation invariant* of the ``LinkedList`` class. The representation invariant of a given class is a predicate that checks whether an instance of the class is well-formed. For example, the representation invariant of the following class:

.. code-block:: java

   public class TimeOfTheDay {
       int h, min, sec;
       ...
   }

could require that ``h`` has a value between 0 and 23, ``min`` has a value between 0 and 59, and ``sec`` has a value between 0 and 59: Only under these constraints an instance of the class ``TimeOfTheDay`` "represents" a correct time of the day. The predicate is said to be an "invariant" because, if the class is correct, it must be always true through all the life of its instances. [2]_

It is possible to implement a representation invariant as a method of its own class, in which case the representation invariant implementation is called a *repOk* method. A repOk method checks whether the instance of the class on which the method is invoked satisfies the representation invariant, returning ``true`` in the positive case and otherwise ``false``. For example, the ``TimeOfTheDay`` class could be equipped by a repOk method as follows:

.. code-block:: java

   public class TimeOfTheDay {
       int h, min, sec;

       boolean repOk() {
           return (0 <= h) && (h <= 23) &&
                  (0 <= min) && (min <= 59) &&
                  (0 <= sec) && (sec <= 59);
       }
       ...
   }

More complex is the definition of a repOk method for the ``LinkedList`` class, because it must scan the sequence of ``LinkedList.Node``\ s starting from ``head``, and determine whether any loop is present:

.. code-block:: java

   public class LinkedList<I> {
       private Node head;

       private static class Node {
           private I value;
           private Node next;
           ...
       }

       boolean repOk() {
           Set<Node> visited = ...;
	   for (Node n = this.head; n != null; n = n.next) {
	       if (visited.contains(n)) return false;
	       visited.add(n)
	   }
	   return true;
       }
       ...
   }

Now let us reconsider the problem of excluding the ill-formed input lists when symbolically executing the ``Target.sum(List)`` method. A possible idea is to exploit the ``jbse.meta.Analysis.assume`` method, and assume at the entry of the ``sum`` method that the ``list`` parameter satisfies its own representation invariant:

.. code-block:: java

   ...
   import static jbse.meta.Analysis.assume;

   public class Target {
       int sum(List<Integer> list) {
           assume(list.repOk());
           ...
       }
   }

Unfortunately this technique, while formally correct, has many practical disadvantages. Since it must completely scan the object it checks to determine its structural correctness, the symbolic execution of a repOk method tendentially accesses all the symbolic references in the input object, causing their resolution. The effect is twofold: First, as discussed at the end of :ref:`ssec-getting-assertions`, it yields an early explosion of the total number of paths; Second, by resolving possibly more symbolic references than the target method would have done, it *overconstrains* the symbolic execution, hindering the generality of the results.

**********
LICS rules
**********

The idea motivating LICS rules is that it is possible to control the shape of the initial input object graph by restraining the possible resolutions of the symbolic references present in it during their (lazy) initialization. This way it is possible to express a surprisingly high variety of structural invariants.

Let's start from the linked list example, and suppose we want to express the assumption that the sequence of list nodes does not contain loops. This can be expressed as a constraint on lazy initialization as follows: When a reference with origin ``{ROOT}:list.head``, ``{ROOT}:list.head.next``, ``{ROOT}:list.head.next.next``... must be resolved, it may be resolved by null or by expansion, but not by alias. We can prove it as follows:

* First, we note that a symbolic reference whose origin is a prefix of another symbolic reference's origin must be resolved before the latter. This implies that ``{ROOT}:list.head`` must be resolved before ``{ROOT}:list.head.next``, which must be resolved before ``{ROOT}:list.head.next.next``, etc.
* The first reference to be resolved must therefore be ``{ROOT}:list.head``: If we forbid its resolution by alias, then it may be resolved only by null, yielding the empty list, or by expansion, yielding a list with (at least) one fresh node. Being fresh, the node cannot be pointed to by any other symbolic reference.
* The second reference to be resolved must be ``{ROOT}:list.head.next``: Since we forbade its resolution by alias it cannot point to the previously assumed node, and it may only be null, yielding a well-formed list with exactly one node, or point to a fresh node, that again cannot be pointed to by any other symbolic reference.
* Since no other symbolic reference in the target program may point to a ``LinkedList.Node``, by recursion we have ensured that exactly one symbolic reference points to a node: In other words, ``list`` does not contain loops.

The previous constraint can be expressed by the following LICS rule::

   {ROOT}:list.head(.next)* aliases nothing

meaning that all the symbolic references whose origins match the regular expression ``{ROOT}:list.head(.next)*`` cannot be resolved by alias.

A comment is necessary. The rule ``<pattern> aliases nothing`` does *not* necessarily mean that a symbolic reference whose origin matches ``<pattern>`` does not share the object it points to with other symbolic references. Let's consider a slight variation of the ``Target`` class:

.. code-block:: java

   public class Target {
       List<Integer> otherList;
       
       int sum(List<Integer> list) {
           ...
       }
   }

In this case the LICS rule ``{ROOT}:list.head(.next)* aliases nothing`` does not constrain, e.g., the symbolic reference with origin ``{ROOT}:this.otherList.head``, that can be freely resolved by alias and point, e.g. to ``{ROOT}:list.head.next``. If we want to rule out the possibility that ``list`` and ``otherList`` share a node, we need to constrain also the origins matching the regular expression ``{ROOT}:this.otherList.head(.next)*``, either by adding another LICS rule::

   {ROOT}:this.otherList.head(.next)* aliases nothing

or by using the following unified rule::

   {R_ANY}.head(.next)* aliases nothing

where ``{R_ANY}`` matches any origin string. This way it is possible to express the constraint that any two distinct lists may not share nodes, a kind of constraint that cannot be expressed by a repOk method.

Now let's consider another variation of the example. This time we add to ``LinkedList.Node``\ s a field ``prev``, that must point to the previous node in the list, or to ``null`` if no previous node exists. This constraint can be expressed as follows::

   {R_ANY}.head.prev expands to nothing
   {R_ANY}.head.prev aliases nothing
   {R_ANY}.head(.next)+.prev not null
   {R_ANY}.head(.next)+.prev expands to nothing
   {R_ANY}.head(.next)+.prev aliases target {$REF}.{UP}.{UP}

These LICS rules have the following meaning:

* The first two rules forbid the symbolic references whose origins match ``{R_ANY}.head.prev`` to be resolved by alias or by expansion: As a consequence, these symbolic references can be only resolved by ``null``, capturing the requirement that, if no previous node exists, the field ``prev`` must contain ``null``.
* The third and fourth rules forbid the symbolic references whose origins match ``{R_ANY}.head(.next)+.prev`` to be resolved by null or expansion, effectively constraining them to be only resolved by alias.
* The fifth rule specifies more precisely how a symbolic reference whose origin matches ``{R_ANY}.head(.next)+.prev`` may be resolved by alias. It states that such a reference must alias the object pointed by the symbolic reference whose origin matches ``{$REF}.{UP}.{UP}``. The latter expression defines a calculation on the origin of the reference to resolve (``{$REF}``), to which we must delete the rightmost field (``{UP}``) twice. In other words::

   {ROOT}:list.head.next.prev must point to {ROOT}:list.head.next.prev.{UP}.{UP} = {ROOT}:list.head
   {ROOT}:list.head.next.next.prev must point to {ROOT}:list.head.next.next.prev.{UP}.{UP} = {ROOT}:list.head.next
   {ROOT}:list.head.next.next.next.prev must point to {ROOT}:list.head.next.next.next.prev.{UP}.{UP} = {ROOT}:list.head.next.next
   ...

(Note that, being ``{$REF}.{UP}.{UP}`` a prefix of ``{$REF}``, the corresponding symbolic reference must have been resolved before the symbolic reference with origin ``{$REF}``, given that ``{$REF}`` has at least two fields.)

Finally, the following LICS rule expresses the fact that ``{ROOT}:list`` can be expanded to a concrete ``LinkedList`` object::

   {ROOT}:list expands to instanceof esecfse2013/LinkedList

*************************
More on the LICS language
*************************
In the previous subsection we introduced the basics of the LICS language. Now we will discuss more in details other aspects of the LICS syntax. The list of all the possible LICS rules is the following::

   <pattern> not null
   <pattern> expands to nothing
   <pattern> expands to instanceof <class>
   <pattern> aliases nothing
   <pattern> aliases instanceof <class>
   <pattern> aliases target <pattern2>
   <pattern> never aliases target <pattern2>

In the previous subsection we introduced all the rules except for ``<pattern> aliases instanceof <class>`` and ``<pattern> never aliases target <pattern>``: The former expresses the constraint that, whenever a symbolic reference whose origin matches ``<pattern>`` is resolved by alias, it must point to an object with class ``<class>``. The latter is a negative version of the ``aliases target`` rule, requiring that, whenever a symbolic reference whose origin matches ``<pattern>`` is resolved by alias, it must not point to an object whose origin matches ``<pattern2>``.

Another important aspect is the syntax of patterns, that slightly differs from that of standard regular expressions. Since the dot (.) character is used to indicate the fields separator, ``{°}`` is used as the pattern matching an arbitrary character. For instance, the pattern that matches any string is ``{°}*``. Similarly, since the dollar ($) character must be used as separator to indicate the class name of nested classes, ``{EOL}`` is used to indicate the end-of-line character. A special pattern syntax can be used in the ``aliases target`` and ``never aliases target`` rules only for the patterns appearing on the right of the ``target`` keyword, i.e., the patterns indicated as ``<pattern2>`` above. These patterns may contain the keyword ``{$REF}``, that as we have seen matches the origin of the reference to resolve, or the keyword ``{$R_ANY}``, that matches the corresponding ``{R_ANY}`` if this is contained in the pattern on the left of the ``aliases`` keyword. Finally, it is possible to prepend the keyword ``{MAX}`` to the pattern on the right of the ``target`` keyword, obtaining a *max-pattern*. A max-pattern ``{MAX}p`` matches the (only) symbolic reference whose origin matches ``p`` and that has the *longest* origin string among all the symbolic references whose origin matches ``p``. A max-pattern is used to indicate the deepest object among a set of objects situated on a given path in the heap. Let us consider the case of a doubly, circular linked list data structure. The rule::

   {R_ANY}.head(.next)* aliases target {MAX}{$R_ANY}.head(.prev)*

expresses the fact that, if we resolve by alias a symbolic reference with origin ``{R_ANY}.head.next.next...next`` (the "rightmost" node) , this must point to the deepest object pointed by the symbolic reference with origin ``{R_ANY}.head.prev.prev...prev`` (the "leftmost" node), therefore correctly closing the circular structure of nodes.



.. [1] Note that JBSE does not determine automatically the list of the concrete types implementing an abstract type (i.e., it does not perform class hierarchy analysis). Therefore, without instructing JBSE about the existence of ``LinkedList`` as a possible concrete subtype of ``List``, JBSE would be unable to resolve ``{ROOT}:list`` by expansion.

.. [2] If the class is mutable, the representation invariant can be temporarily violated *while* a mutator method is changing the state of an instance. But it is required that at the end of the execution the mutator method leaves the object in a state where the invariant holds true, and that all the intermediate states traversed by the object while the mutator executes are externally invisible.
