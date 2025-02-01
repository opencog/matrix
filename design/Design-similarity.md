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
 2) Given an edge, get the left and right vertexes.
 3) Given the matrix, get left and right wild-cards (marginals; edges
    with the left or right vertexes replaced by wild-cards).
 4) A list of all edges.
 5) Ability to get all edges from a StorageNode
 6) Delete all edges (and the correspodning entries in a StorageNode)

This API is documented in `object-api.scm` and a concrete example
appears in `edge-pairs.scm`. The file `object-api.scm` also defines
the `add-pair-stars` API, which, given the above, adds decorations
that almost all otherlayers above need. These are:

 7) Lists of the left and right basis elements. The left basis is the
    set. `{x | (x,y) exists in the atomspace for some y}`
 8) Size of the left and right basis
 9) True/false membership predicates: does an Atom appear in the left
    or right basis set?
 10) List of the left and right duals to a given Atom. The left dual
    for a given Atom Y is `{x | (x,Y) given fixed Y}`

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

For item (3), 
