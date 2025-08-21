This website was way more complicated than the website in the previous level.
The website in this level was a chatroom that again we needed to use to inject JS code that will send an alert.

when reading `post-store.js` I saw that all the posts were (again) not vetted or formatted.
```javascript
function Post(message) { 
  this.message = message;
  this.date = (new Date()).getTime();
}

// ... function PstDB
this.save = function(message, callback) {
    var newPost = new Post(message);
    var allPosts = this.getPosts();
    allPosts.push(newPost);
    window.localStorage["postDB"] = JSON.stringify({
      "posts" : allPosts
    });
 
    callback();
    return false;
  }
```

Now I just wanted to take a look at the piece of code that displays the posts.
```javascript
function displayPosts() {
	var containerEl = document.getElementById("post-container");
	containerEl.innerHTML = "";

	var posts = DB.getPosts();
	for (var i=0; i<posts.length; i++) {
	  var html = '<table class="message"> <tr> <td valign=top> '
		+ '<img src="/static/level2_icon.png"> </td> <td valign=top '
		+ ' class="message-container"> <div class="shim"></div>';

	  html += '<b>You</b>';
	  html += '<span class="date">' + new Date(posts[i].date) + '</span>';
	  html += "<blockquote>" + posts[i].message + "</blockquote";
	  html += "</td></tr></table>"
	  containerEl.innerHTML += html; 
	}
}
```

So I saw that without any formatting it just puts the post in the html without testing. so like in level #1 I tried putting directly a script tag
```html
<script>alert()</script>
```
this did not work but it did put up an empty message. This meant that the html was indeed injected to the website but it was not running. So I wanted to try and put a button that will trigger this alert instead.

```html
<a onClick="alert()">Click Me!</a>
```
Doing this solved the challenge and allowed me to move to level #3!