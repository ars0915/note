- Read Template
  template:: Read Template
  template-including-parent:: false
	- book:: <%setinput: BookTitle%>
	  author:: <%setinput: Author%>
	  tags:: <%setinput: Tags%>
-
- Chapter Template
	- book:: <%setinput: BookTitle%>
	  tags:: <%setinput: Tags%>
-
- Quota Template
  template:: Quote Template
  template-including-parent:: false
	- 💬 "<%setinput: QuoteText%>"
	  author:: <%setinput: Author%>
	  topic:: "<%setinput: Topic%>"
-
-