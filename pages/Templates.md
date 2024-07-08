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
  template:: Reference/Paper Template
  template-including-parent:: false
	- <%setinput: Author%> "<%setinput: Title%>," in *<%setinput: ConfName%>*, <%setinput: Time%>, pp.
	  type:: [[Paper]]
- Web Page Template
	- <%setinput: Author%>, "<%setinput: Title%>," *<%setinput: WebPageName%>*, Source/production information, [[Date of internet publication]]. [Format: Online]. Available: [link_to_page](link_to_page). [Accessed: [[Date of access]]].
	  	  template:: Reference/Web Page 
	  	  type:: [[Web Page]]