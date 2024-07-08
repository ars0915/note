- Read Template
  template:: Read Template
  template-including-parent:: false
	- public:: true
	  author:: <%setinput: Author%>
	  tags:: <%setinput: Tags%>
	  status:: <%setinput: Status%>
	- ## Chapter
		- {{query (property book <%setinput: BookTitle%>)}}
-
- Chapter Template
  template:: Chapter Template
  template-including-parent:: false
	- book:: <%setinput: BookTitle%>
	  tags:: <%setinput: Tags%>
	  chapter:: <%setinput: Chapter%>
-
- Quota Template
  template:: Quote Template
  template-including-parent:: false
	- 💬 "<%setinput: QuoteText%>"
	  author:: <%setinput: Author%>
	  topic:: "<%setinput: Topic%>"
-
- ## Reference Template
- Paper Template
  template:: Paper Template
  template-including-parent:: false
	- <%setinput: Author%> "<%setinput: Title%>," in *<%setinput: ConfName%>, <%setinput: Time%>, pp.