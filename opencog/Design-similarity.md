Sesign notes for similarity measures
====================================
Design notes and ideas for similarity measures. Attempts to get at the
core problem that the matrix API was invented to solve. Tries to do this
at a deeper, more abstract and simpler way.

Motivation
---------
The current matrix API provides an extensive set of objects and
functions for working with sparse vector embeddings. The vast selection
is characteristic of what human programmers wnt and expect when
attempting to write code to solve some problem. However, that same
richness and variety impedes automatition and automatic construction.

Almost all of the matrix API was developed to support one generic
problem: obtaining the similarity between two things. Thus, the task at
hand is find some simpler, easier, more basic way of thinking about and
working with similarity.
