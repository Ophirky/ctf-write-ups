While reading the source code of the level I saw that the code is directly running scripts without validating them and knew that that is where I need to inject my malware.

```js
var scriptEl = document.createElement('script');
      var scriptEl = document.createElement('script');
      
      /* SOME OTHER CODE */
      
      scriptEl.src = url;
```

After finishing the code review I understood that since it is just taking a JS file from the URL I need to host an malicious JS file that will alert. 

```js
function getGadgetName() { 
  return window.location.hash.substr(1) || "/static/gadget.js";
}
```

Since I couldn't host one on my own I used the data tag in the URL that tells the url that a file is transferred.
on top of this I could not use the `http://` or `https://` since there was a test eliminating all URLS that started like that.

```js
// This will totally prevent us from loading evil URLs!
if (url.match(/^https?:\/\//)) {
	setInnerText(document.getElementById("log"),
	  "Sorry, cannot load a URL containing \"http\".");
	return;
}
```

luckily you don't really need it.

at the end this was the injected url:
https://xss-game.appspot.com/level6/frame#data:text/javascript,alert()