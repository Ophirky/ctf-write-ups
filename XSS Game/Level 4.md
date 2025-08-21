At first glance I immediately tried to just put a script tag in the timer spot but that did not work

after looking deeper at the code I saw that it was using a local variable saved in the server accessed using jinja.
looking at the network tab I could see that the request is being sent via the URL directly and is encoded.

Figuring this out I decided to inject the code directly via the URL
after trying this line it did not work..
`https://xss-game.appspot.com/level4/frame?timer=')alert()`


now I wanted to look deeper at the start timer function and saw that the timer variable was directly put in.

I tried putting a ' in the form input and see what happens
and the following error appeared in the console
`Uncaught SyntaxError: missing ) after argument list`
which means that the js function call was never closed. meaning I could try and inject something there. this meant that I could escape the function and insert code of my own.

with this in mind I tried inserting the following injection:
`'); alert(); '`

after this I took a closer look at the code and understood that I needed to escape the apostrophe at the end of the function call so I injected `'); alert('`

```html
<img src="/static/loading.gif" onload="startTimer('{{ timer }}');"/>
<img src="/static/loading.gif" onload="startTimer(''); alert('');"/>
```

solving the level