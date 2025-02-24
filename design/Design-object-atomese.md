Design Notes - Objective Atomese
================================
Design thoughts about an object-oriented infrastructure, in Atomese.
nothing here yet, except for a sketch of ideas.

The below sketches ideas (circa 2024) for an object-oriented variant of
Atomese. It seems like it would not be hard to implement; it extends
conventional Atomese in a minor way. Yet, it also feels insufficiently
abstract. Over time, its become clear that the `ExecutionOutputLink` and
`EvaluationLink` are problematic. They work, in the sense they were
originally imagined to work, but they break up data flow in awkward
ways. They provide the wrong kind of abstraction for an agentic
architecture. Thus, see [Design-similarity](Design-similarity.md)
instead.

Motivation
----------
The primary goal is to replicate some fraction of the code in the
[matrix](../matrix) directory, but to do so in pure
[Atomese](https://wiki.opencog.org/w/Atomese), instead of scheme. The
matrix code has proven to be exceptionally useful and powerful, but there
is now a need to be able to run it in (parallel) pipelines (configured
with [ProxyNode](https://wiki.opencog.org/w/ProxyNode)s.) This includes
running it over the network (via
[CogStorageNode](https://wiki.opencog.org/w/CogStorageNode)) or otherwise
interacting with [StorageNode](https://wiki.opencog.org/w/StorageNode)s
in interesting, non-trivial ways. For that, the matrix code must become
specifiable in Atomese. (Yes, we could just wrap it in a
[GroundedProcedureNode](https://wiki.opencog.org/w/GroundedProcedureNode)
but that would be ugly and counter-productive.)

Object Basics
-------------
The actual code in matrix is written in scheme, and uses a peculiar OO
programming style variously called
[Generic Programming](https://en.wikipedia.org/wiki/Generic_programming),
[Parametric Polymorphism](https://en.wikipedia.org/wiki/Parametric_polymorphism)
or [Traits](https://en.wikipedia.org/wiki/Trait_(computer_programming)).
In short, it is an OO style similar to that of JavaScript, rather than of
C++.


How might this work in Atomese?  Maybe like this:
```
(DefineLink
	(DefinedMethodLink
		(PredicateNode "some object classname")
		(PredicateNode "some method"))
	(LambdaLink
		(VariableList ...)
		... atomese ...))
```
The `DefinedMethodLink` isn't really needed; one could just use a
`ListLink` and this would work just fine, today. But it's convenient to
have a special link type devoted to this task.

Invocation of the object-instance+method pair is "just like always", with
the [ExecutionOutputLink](https://wiki.opencog.org/w/ExecutionOutputLink):
```
(cog-execute!
	(ExecutionOutputLink
		(DefinedMethodLink
			(PredicateNode "some object classname")
			(PredicateNode "some method"))
		(ListLink
			(Predicate "some object instance")
			... method args ...)))
```

The above more or less almost works today, and only minor tweaks are needed
to get this fully working.  That's because there is no particularly great
secret to OO programming: just provide a way to pass the class instance as
the first, "this" argument. That's all.


Pipelines
---------
The need for the above is driven by the need to be able to specify
processing pipelines, using the
[DynamicDataProxyNode](https://wiki.opencog.org/w/DynamicDataProxyNode)
This [ProxyNode](https://wiki.opencog.org/w/ProxyNode) causes Atomese
snippets to run, whenever a specific key is used to get the Value on
any Atom handled by the Proxy. The prototypical example is to compute
the mutual information (MI) on some matrix element.

Matrix elements often (but not always) have the form
```
(EvaluationLink
	(Predicate "name of some matrix")
	(List
		(Atom "left or row index")
		(Atom "right or column index")))
```
Some kinds of matrix have a much more complex form; however, in general,
they all are triplets of the form `(root, left, right)`. In order to be
able to do generic matrix computations, there must be some way of getting
the left and right elements given the triplet, and sometimes to construct
the triplet, given the left and right elements. But how?

In the scheme matrix code, each object has methods, such as
`get-left-element` and `get-right-element` and so on, that will obtain the
needed item for the given matrix instance. To do the same thing in Atomese,
we need to have a way of specifying method names, and using them.

Thus, ***every*** Atomese matrix object ***must*** define the following
methods:
* `(Predicate "*-left-element-*")` -- return the left element
* `(Predicate "*-right-element-*")` -- return the right element
* And many more; See the [object-api.scm](../matrix/object-api.scm] file
  for the starter set.

Example
-------
Lets assume the simplest case, where the matrix is in the form of the
`EvaluationLink` shown above. One would then have:

```
(DefineLink
	(DefinedMethodLink
		(Predicate "*-Basic Pairs Matrix-*")
		(Predicate "*-left-element-*"))
	(Lambda
		(Variable "$matrix-entry") ; This will be the EvaluationLink;
		(ElementOf
			(ElementOf (Variable "$matrix-entry") (Number 1)) ; the ListLink
			(Number 0))  ; first item in the ListLink.
	))
```

To run this:
```
(cog-execute!
	(ExecutionOutputLink
		(DefinedMethodLink
			(Predicate "*-Basic Pairs Matrix-*")
			(Predicate "*-left-element-*"))
		(EvaluationLink
			(Predicate "word pairs matrix")
			(List
				(Word "blue")
				(Word "sky")))))
```
This would return `(Word "blue")` upon execution.


This is perhaps a terrible example.  We could (maybe should) use
FilterLink to do this work.

```
(DefineLink
	(DefinedMethodLink
		(Predicate "*-Basic Pairs Matrix-*")
		(Predicate "*-left-element-*"))
	(Lambda
		(Variable "$matrix-entry") ; This will be the EvaluationLink;
		; The FilterLink will extract the left elt.
		(FilterLink
			(RuleLink
				;; Variables in the pattern
				(VariableList
					(Variable "$pred") (Variable "$left") (Variable "$right"))
				;; The matrix pattern to match.
				(Evaluation
					(Variable "$pred")
					(List (Variable "$left") (Variable "$right")))
				;; The rewrite -- what to return, after pattern matching.
				(Variable "$left"))
			(Variable "$matrix-entry"))))
```

Also, this form is possible:
```
(DefineLink
	(DefinedMethodLink
		(Predicate "*-Basic Pairs Matrix-*")
		(Predicate "*-left-element-*"))
	(Lambda
		(Variable "$matrix-entry") ; This will be the EvaluationLink;
		; The UnifierLink will extract the left elt.
		(UnifierLink
			(Lambda
				;; Variables in the pattern
				(VariableList
					(Variable "$pred") (Variable "$left") (Variable "$right"))
				;; The matrix pattern to match.
				(Evaluation
					(Variable "$pred")
					(List (Variable "$left") (Variable "$right"))))

			;; What to unify against
			(Variable "$matrix-entry")

			;; The unifer re-write -- What to return, after pattern matching.
			(Variable "$left"))))
```

We can get rid of the unused, un-needed variables:

```
(DefineLink
	(DefinedMethodLink
		(Predicate "*-Basic Pairs Matrix-*")
		(Predicate "*-left-element-*"))
	(Lambda
		(Variable "$matrix-entry") ; This will be the EvaluationLink;
		; The FilterLink will extract the left elt.
		(Filter
			(Rule
				;; Variable that we will bind.
				(Variable "$left")
				;; The matrix pattern to match.
				(Evaluation
					(Sign 'PredicateNode)
					(List (Variable "$left") (Sign 'WordNode)))
				;; The rewrite -- what to return, after pattern matching.
				(Variable "$left"))
			(Variable "$matrix-entry"))))
```

All four of these forms should "just work".

NB SignNode is a synonym for (SignatureLink (TypeNode 'foo))


TODO
----
* Handle SignLink See issue #2602
* Handle Signatures in the pattern matcher, plus examples & tests.
* Change BoolValue so it can create maskes from bit-specs!?
* Change NumberOfLink so it can convert bool masks to element numbers ??
* Most FunctionLinks need to be able to handle streams i.e. QueueValues.
