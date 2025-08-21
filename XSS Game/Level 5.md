First I started by exploring and quickly cam to conclusion that the input is not relevant. It is not used or sent anywhere.

Once I started looking at the code I saw that the only variable used is the "next" variable. 
I saw that after every page transition the next variable changes.
When entering the welcome page it is set manually to "confirm" only when transferring to the sign up page.
```html
<a href="/level5/frame/signup?next=confirm">Sign up</a>
```

That meant that I could set the next variable to escape the string and inject the alert statement.
so I tried going to this URL:
`https://xss-game.appspot.com/level5/frame/signup?next=confirm"%20onclick="alert()"`

since that did not work (obviously) then I decided to map in what page next equals what:
- signup.html - next=confirm (client side)
- welcome.html - next=None
- confirm.html - next=welcome (server side)

now what each page makes next when leaving:
- welcome - next=confirm (client side)
- signup - next is not changed
- confirm - next=welcome (server-side)

knowing this I knew that I needed to where next is used in the client side.
in the confirm.html page next was used to transfer the user a page after 5 seconds. this was done by directly inserting next into the window.location variable.
```html
<script>
      setTimeout(function() { window.location = '{{ next }}'; }, 5000);
</script>
```

now I wanted to try and see if I could break this to escape the string.
Now i have decided to take a hint: "The title of this level is a hint."
The title of the level was "Breaking Protocol". This meant that I was on the right track.

ok. Now that I knew that I was on the right track I researched a way to execute JS code via the URL and found that there is a tag called javascript that I could try and put within the URL.
`http://example.com/index.html?javascript:alert()`

but that did not work so I continued with my research and saw that it worked the following way
`javascript:alert()`

so I created this URL:
`https://xss-game.appspot.com/level5/frame/signup?next=javascript:alert()`

Level Solved.





