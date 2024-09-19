public:: true
tags:: Algorithm, Graph

- ## 題目
- graph undirected, number from 1 to N and M edges
  described by two arrays, A and B both of length M
  A pair (A[K], B[K]), for K from 0 to M-1, describes an edge between vertex A[K] and vertex B[K]
- assign all values from the range [1..N] to the vertices of the graph, giving one number to each of the vertices. Fo it in such a way that the sum over all edges of the values at the edges' endpoints is maximal
- i.e.
  N=5, A=[2,2,1,2],B=[1,3,4,4]n the graph has for edges: (2,1_,(2,3),(1,4),(2,4). in order to obtain the maximum sum of weight, you can assign the following values to the vertices : 3,5,2,4,1
  This way we obtain the sum of values at all edges' endpoints equal to 7+8+7+9=31