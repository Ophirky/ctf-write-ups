This level contained a "datacenter" like website that had three images that you could toggle through.

Looking at the HTML code of this page I could see that to access the images the website required a serial number that was not vetted or tested. I wanted to use this to inject the JS alert code.
```javascript
function chooseTab(num) {
	// Dynamically load the appropriate image.
	var html = "Image " + parseInt(num) + "<br>";
	html += "<img src='/static/level3/cloud" + num + ".jpg' />";
	$('#tabContent').html(html);

	window.location.hash = num;
	// rest of code...
```

After reading this the only place I could inject my code was in the URL.
First I tried closing the `<img>` tag and putting a script tag but that did not work properly:
`https://xss-game.appspot.com/level3/frame#/><script>alert()</script>`

but the image did not load since it did not get a valid number.
So now I could use the `onerror` argument and make it alert if it can't load the image
`https://xss-game.appspot.com/level3/frame#1'onerror='alert()'>`

which solved level #3!