- Read Template
  template:: Read Template
  template-including-parent:: false
	- public:: true
	  author:: <%setinput: Author%>
	  tags:: <%setinput: Tags%>
	  status:: <%setinput: Status%>
	- {{query (and (property chapter) (and <% current page %>))}}
-
id:: 668bcac4-0497-4ea8-8e15-05190313d1a5
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
-
- Algorithm Template
  template:: Algorithm Template
  template-including-parent:: false
	- public:: true
	  tags:: <%setinput: Tags%>
	- ## 題目
	- ##
-
- ## Reference Template
- Paper Template
  template:: Reference/Paper Template
  template-including-parent:: false
	- <%setinput: Author%> "<%setinput: Title%>," in *<%setinput: ConfName%>*, <%setinput: Time%>, pp.
	  type:: [[Paper]]
- Web Page Template
  template:: Reference/WebPage Template
  template-including-parent:: false
	- <%setinput: Author%>, "<%setinput: Title%>," *<%setinput: WebPageName%>*, Available: [link_to_page](<%setinput: Link%>). 
	  type:: [[Web Page]]
-
-