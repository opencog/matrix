Sesign notes for similarity measures
====================================
Design notes and ideas for similarity measures. Attempts to get at the
core problem that the matrix API was invented to solve. Tries to do this
at a deeper, more abstract and simpler way.

Motivation
----------
The current matrix API provides an extensive set of objects and
functions for working with sparse vector embeddings. The vast selection
is characteristic of what human programmers wnt and expect when
attempting to write code to solve some problem. However, that same
richness and variety impedes automation and automatic construction.

Almost all of the matrix API was developed to support one generic
problem: obtaining the similarity between two things. Thus, the task at
hand is find some simpler, easier, more basic way of thinking about and
working with similarity.

(Vector) Embeddings
-------------------
All things have a local neighborhood of other things that are connected
and related to them. They have properties, aspects, relationships to
other nearby things. This embedding can be (mathematically) abstracted
in several ways:

* The "thing" is a vertex in a graph, and (labelled) graph edges
  connect it to other "things". This automatically defines the local
  neighborhood: anything that is only one edge away is necessarily local.

* Edges may be assigned a real number (floating point) weight, giving a
  more refined sense of distance.

* Same as above, but with hyperedges and hypergraphs.

* Graph theory notes that every graph has a corresponding adjacency
  matrix. The rows and columns of this matrix are vectors, from here on
  called the 'embedding vectors'. These vectors are sparse (not all
  vertices are connected by edges), and usually extremely sparse
  (because scale-free networks are extremely sparse, and most things
  in nature are scale-free.)

These last four bullets lead to the existing matrix API. The network
graph is "already there", in the AtomSpace; the edges are clearly
marked, the embedding vectors are already implicit in the graph, and
all that was needed was an API to work with these (sparse) vectors.
Given a vector API, a cornucopia of similarity measures can be created.

Review of the existing API
--------------------------
The original matrix API was invented to work with word-pairs. Each word
is a vertex; the edge between them forms a word-pair. Given a collection
of edges, one could think of this as forming an adjacency matrix, with
the edges being the non-zero entries in the matrix. The edges are
directed, so the matrix is not symmetric. Any given row or column in the
matrix is a sparse vector. Given a collection of vectors, there is a
rich set of mathematical operations that can be applied to them: dot
products, mutual information, Jacquard and Hamming distances, etc.

To implement the above, the original matrix API used an OO design style
called "parametric polymorphism", defining a base class API, that all
higher analysis layers could be stacked on top of. The base class
consisted of these parts:

 1) Get the types of the left and right vertexes, and the edge.
    (The `'left-type`, `'right-type` and `'pair-type` methods.)
 2) Given an edge, get the left and right vertexes.
    (The `'left-element` and `'right-element` methods.)
 3) Given the matrix, get left and right wild-cards (marginals; edges
    with the left or right vertexes replaced by wild-cards).
 4) A list of all edges.
 5) Ability to get all edges from a StorageNode
 6) Delete all edges (and the corresponding entries in a StorageNode)

This API is documented in `object-api.scm` and a concrete example
appears in `edge-pairs.scm`. The file `object-api.scm` also defines
the `add-pair-stars` API, which, given the above, adds decorations
that almost all otherlayers above need. These are:

 7) Lists of the left and right basis elements. The left basis is the
    set. `{x | (x,y) exists in the atomspace for some y}` (Provided by
    the `'left-basis` and `'right-basis` methods.)
 8) Size of the left and right basis
 9) True/false membership predicates: does an Atom appear in the left
    or right basis set? (These are provided by the `'in-left-basis?`
    and `'in-right-basis?` methods.)
 10) List of the left and right duals to a given Atom. The left dual
    for a given Atom Y is `{x | (x,Y) given fixed Y}` (This is provided
    by the `'left-duals` and `'right-duals` methods.)

The "problem" with this API is that it is implemented in scheme, and
all of the data is stored in scheme structures. This is a fatal flaw,
and probmpts these re-design notes.

Queries as Tensors
------------------
It appears that the original API can be completely replaced and
generalized by using MeetLink, and generalized by using RuleLink and/or
QueryLink. In retrospect, this is "obvious", and I'm quite mystified why
I didn't do it this way from the beginning. Perhaps it just wasn't so
obvious at first. Let's spell out the obvious.

The current word-pair representation for the pair (some, thing) is:
```
   (Edge (Predicate "word-pair") (List (Word "some") (Word "thing")))
```
All such edges can be found by executing the query
```
   (Query
      (VariableList
         (TypedVariable (Variable "$left") (Type 'Word))
         (TypedVariable (Variable "$right") (Type 'Word)))
      (Present
         (Edge (Predicate "word-pair")
            (List (Variable "$left") (Variable "$right"))))
      (Edge (Predicate "word-pair")
         (List (Variable "$left") (Variable "$right"))))
```
Running this query provides item (4) above. The query results are cached
on the query itself, using itself as the key. That is, `(cog-value q q)`
will return the query results from the most recent execute of Query `q`.

Obviously, the above can be replaced by "any" query at all. The current
rules seem to require explicit variable declarations, so that the types
become visible for item (1).

For item (3), the `'left-wildcard` method takes a single `ITEM` and
returns
```
   (Edge (Predicate "word-pair")
      (List (Any "left-wild") ITEM))
```
The obvious "solution" is to replace this with
```
   (Edge (Predicate "word-pair")
      (List (Variable "$left") ITEM))
```
The preceeding questions need to be "is this really needed?" and "what
is this used for?" The answer seems to be is that these are used for
storing marginals, only.

This then begs the question: "what do we want, for marginals"? Ther are
several plausible answers. Consider the general tensor query:
```
   (Query
       (VariableList VAR-1 VAR-2 ... VAR-N)
       (AndLink PATTERN-TERM-1 PATTERN-TERM-2 ... PATTERN-TERM-K)
       REWRITE-1
       REWRITE-2
       ...
       REWRITE-M
```
One answer is to claim that one wants marginals for the search pattern,
given by the `AndLink`. For N=2 variables, call them 'left' and 'right',
there are only two marginals, obtained by plugging into search pattern.
For larger N, there are N-factorial marginals.

In this case, one asks "what's the point of the rewrites?". The answer
here is that we actually want marginals on the rewrites, and not on the
search pattern. For N=2 and M=1 we regain the original word-pair matrix.
In that original matrix, the only rewrite was identical to the search
pattern, thus a confusion about which is which.

The rewrites provide an opportunity to specify exactly which marginals
one is interested in. Thus, for example, the original matrix, with
wildcards, can be described as:
```
   (Query
      (VariableList
         (TypedVariable (Variable "$left") (Type 'Word))
         (TypedVariable (Variable "$right") (Type 'Word)))
      (Present
         (Edge (Predicate "word-pair")
            (List (Variable "$left") (Variable "$right"))))
      (Edge (Predicate "word-pair")
         (List (Variable "$left") (Variable "$right")))
      (Edge (Predicate "word-pair")
         (List (Any "left-wild") (Variable "$right")))
      (Edge (Predicate "word-pair")
         (List (Variable "$left") (Any "right-wild"))))
```
This has three rewrites: the diagonal, and the left and right marginals.
This has several nice properties. One is that, if the diagonal is not
required, then it can be omitted from the rewrites. This will save
RAM associated with the query result cache, expecially if it is large.

The down-size is the marginals aren't auto-generated. Lacking a specific
use-case example, its hard to see what kind of solution could be
provided. The generic case seems to work well.

Item (2) could be done with a `FilterLink` plus `RuleLink` combo. For
the word-pair matrix, the left-element axtractor would be
```
   (Filter
      (Rule
         (VariableList
            (TypedVariable (Variable "$left") (Type 'Word))
            (TypedVariable (Variable "$right") (Type 'Word)))
         (Edge (Predicate "word-pair")
            (List (Variable "$left") (Variable "$right")))
         (Variable "$left"))
      (List
         (Edge (Predicate "word-pair")
            (Word "some") (Word "thing"))))
```
Notable aspects to this structure:
* The search pattern is specified with a `Rule` instead of a `Query`.
  That's OK, given that Rule is a base class for Query. In practice,
  its a bit confusing.
* The `Rule` doesn't include the `PresentLink` (more generally, the
  `AndLink`). But this is a historical design accident...
* Should the `FilterLink` be amendeded to use the existing query-engine
  infrastructure, but limit the search space to the input vector? Maybe,
  but there seems to be little utility in this, since back-tracking will
  never be required. Filtering is a lot easier than querying. So, no,
  the query engine should not be used for filtering.
* Filtering should be unit tested for correctness on `PresentLink`, etc.
  in the search pattern. Not clear this is fully supported at this time!?
 
Anyway, the above replaces item (2) with a generic rewrite, which is a
good thing, I guess.

Is Item (2) actually needed? Searching the code base, then answer seems
to be "no". The method only shows up in implementations of derived
clases. So I think this can be scratched.

Item's (5) and (6) are already in the StorageNode API, although this API
also needs to be converted to pure Atomese.

Item (7): This is already cached in the query search results!

Item (8): Use SizeOfLink

Item (9): Does not appear to be used anywhere in the code.

Item (10): ... Almost unused; the `learn` code uses this in one place
for set subtraction, and another place for agglomerative MI.

### Conclusion
Simply using the `QueryLink` and caching the rewrite results is
sufficient for use to replace the parametric polymorphism base design.
Nothing more needs to be done, here. (Except possibly more testing,
and moving the search result keys to the rewrites from the query
itself.)

Compute processing
------------------
The matrix code has two parts: first, the specification of the base
matrix, and second, the computation of the assorted similarity measures,
given the base matrix.

The above deals with the base. What about the similarities?
