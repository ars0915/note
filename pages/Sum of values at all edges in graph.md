public:: true
tags:: Algorithm, Graph

- ## 題目
	- graph undirected, number from 1 to N and M edges
	  described by two arrays, A and B both of length M
	  A pair (A[K], B[K]), for K from 0 to M-1, describes an edge between vertex A[K] and vertex B[K]
	- assign all values from the range [1..N] to the vertices of the graph, giving one number to each of the vertices. Fo it in such a way that the sum over all edges of the values at the edges' endpoints is maximal
	- i.e.
	  N=5, A=[2,2,1,2],B=[1,3,4,4]n the graph has for edges: (2,1),(2,3),(1,4),(2,4). in order to obtain the maximum sum of weight, you can assign the following values to the vertices : 3,5,2,4,1
	  This way we obtain the sum of values at all edges' endpoints equal to 7+8+7+9=31
- ## Problem Breakdown
	- **Graph Representation**: The graph is described by two arrays `A` and `B`, where each pair `(A[K], B[K])` represents an edge between vertices `A[K]` and `B[K]`.
	- **Objective**: Assign distinct values from 1 to N to the vertices in such a way that the sum of the values at the endpoints of all edges is maximized.
- ## Approach
	- **Degree Calculation**: 
	  Calculate the degree of each vertex (i.e., the number of edges connected to that vertex). The degree of a vertex is important because vertices with higher degrees will contribute more to the total sum if they are assigned higher values.
	- **Greedy Assignment**:
	  Assign higher values to vertices with higher degrees to maximize their contribution to the sum of edge weights.
- ### Steps
	- **Compute Degrees**: Calculate the degree of each vertex.
	- **Sort Vertices**: Sort vertices based on their degrees in descending order.
	- **Assign Values**: Assign the highest available value to the vertex with the highest degree, the second highest value to the vertex with the second highest degree, and so on.
- ## Code
	- ```go
	  package main
	  
	  import (
	      "fmt"
	      "sort"
	  )
	  
	  type Vertex struct {
	      id    int
	      degree int
	  }
	  
	  // Function to calculate the maximum sum of values at the endpoints of all edges
	  func Solution(N int, A []int, B []int) int {
	      // Step 1: Compute the degree of each vertex
	      degrees := make(map[int]int)
	      for i := 0; i < len(A); i++ {
	          degrees[A[i]]++
	          degrees[B[i]]++
	      }
	      
	      // Convert degrees to a slice of Vertex
	      vertices := make([]Vertex, 0, N)
	      for i := 1; i <= N; i++ {
	          vertices = append(vertices, Vertex{id: i, degree: degrees[i]})
	      }
	      
	      // Step 2: Sort vertices by degree in descending order
	      sort.Slice(vertices, func(i, j int) bool {
	          return vertices[i].degree > vertices[j].degree
	      })
	      
	      // Step 3: Assign values from N down to 1
	      vertexValue := make(map[int]int)
	      value := N
	      for _, vertex := range vertices {
	          vertexValue[vertex.id] = value
	          value--
	      }
	      
	      // Step 4: Calculate the total sum of values at the endpoints of all edges
	      totalSum := 0
	      for i := 0; i < len(A); i++ {
	          totalSum += vertexValue[A[i]] + vertexValue[B[i]]
	      }
	      
	      return totalSum
	  }
	  
	  func main() {
	      N := 5
	      A := []int{2, 2, 1, 2}
	      B := []int{1, 3, 4, 4}
	      fmt.Println(Solution(N, A, B)) // Output: 31
	  }
	  ```
	  
	  **Degree Calculation**: We calculate the degree of each vertex and store it in a map.
	  **Sorting**: We sort the vertices based on their degrees in descending order.
	  **Value Assignment**: We assign the highest available values to vertices with the highest degrees.
	  **Sum Calculation**: We compute the total sum of values at the endpoints of all edges.
- ## Complexity:
	- **Time Complexity**: The algorithm primarily involves calculating degrees, sorting the vertices, and summing up the values. The overall time complexity is `O(M+Nlog⁡N)O(M + N \log N)O(M+NlogN)`, where MMM is the number of edges and NNN is the number of vertices.
	- **Space Complexity**: The space complexity is `O(N)O(N)O(N)` for storing degrees and vertex values.