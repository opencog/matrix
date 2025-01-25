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
richness and variety impedes automatition and automatic construction.

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

* The "thing" is a vertex in a graph, and (labelled) graph edges connect
  it to other "things". This automatically defines the local
  neighborhood: anything that is only on edge away is necessarily local.

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
