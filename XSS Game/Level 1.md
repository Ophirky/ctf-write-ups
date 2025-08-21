The goal here was to make the website execute JS code using the input line in the given website.

Viewing the code of the website I saw that when I sent  a query that did not have any results it would just format my query into the page without checking anything (line 45 in level.py).
```python
message = "Sorry, no results were found for <b>" + query + "</b>."
```

seeing this I just tried entering an input of a script tag containing an alert
```html
<script>alert()</script>
```
Doing this accomplished the challenge.