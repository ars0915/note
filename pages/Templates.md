- Read Template
  template:: Read Template
  template-including-parent:: false
	- author:: <%setinput: Author%>
	  tags:: <%setinput: Tags%>
	  status:: <%setinput: Status%>
	- ## Chapter
		-
-
- Chapter Template
  template:: Chapter Template
  template-including-parent:: false
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