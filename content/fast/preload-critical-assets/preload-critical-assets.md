---
title: Preload critical assets to improve loading speed
author: houssein
page_type: glitch
glitch: preload-critical-assets
---

In this codelab, the performance of the following web page will be improved
by preloading and prefetching a few resources:

![App Screenshot](./san-francisco.png)

## Measure

First measure how the website performs before adding any optimizations (see also
Discover performance opportunities with Lighthouse):

1. Click on the Show Live button to boot up the app.

![Show live button](./show-live.png)

1. Open DevTools by pressing `CMD + OPTION + i` / `CTRL + SHIFT + i`.
2. Click on the **Audits** panel.
3. Select the **Performance** checkbox.
4. Click **Run Audits** to generate a report.

Lighthouse shows the following failed audit for a resource that is fetched
late:

![image](./preload-requests-codelab.png)

Open the **Network** panel in DevTools and take a look at all the resources that
are fetched.

![Network panel without any preloading](network-panel-one.png)

The `main.css` file is not fetched by a Link element (`<link>`) placed in the HTML
document, but a separate JavaScript file, `fetch-css.js`, attaches
the Link element to the DOM after the `window.onLoad` event. This means that the
file is only fetched _after_ the browser finishes parsing and executing the JS
file.  Similarly, a web font (`K2D.woff2`) specified within `main.css` is only
fetched once the CSS file has finished downloading.

The **critical request chain** represents the order of resources that are
prioritized and fetched by the browser. For this web page, it currently looks
like this:

```bash
├─┬ / (initial HTML file)  
  └── fetch-css.js
    └── main.css  
      └── K2D.woff2
```

Since the CSS file is on the third level of the request chain, Lighthouse has
identified it as a late-discovered resource. 

## Preload critical resources

The `main.css` file is a critical asset that's needed immediately as soon as the
page is loaded. For important files like this resource that are fetched late in
your application, use a link preload tag to inform the browser to download
it sooner by adding a Link element to the head of the document.

Add a preload tag for this application:

<pre class="prettyprint">
&lt;head&gt;
  &lt;!-- ... --&gt;
  <strong>&lt;link rel=&quot;preload&quot; href=&quot;main.css&quot; as=&quot;style&quot;&gt;</strong>
&lt;/head&gt;
</pre>

The `as` attribute is used to identify which type of resource is being
fetched, and `as="style"` is used to preload stylesheet files. 

Reload the application and take a look at the **Network** panel in DevTools.

![Network panel after preloading](./network-panel-two.png)

Notice how the browser fetches the CSS file before the JavaScript
responsible for fetching it has even been finished parsing. With preload, the browser
knows to make a preemptive fetch for the resource with the assumption that it is
critical for the web page.

<div class="aside note">
If this was a real production app, it would make more sense to just place
a <code>&lt;link&gt;</code> element in <code>index.html</code> to fetch the CSS file 
instead of using JavaScript to append it. Browsers already know to fetch a CSS file
defined at the head of an HTML document with a high priority as soon as possible. 
However, preload is used in this codelab to demonstrate the best course of action for files that
are fetched late in the request chain. For a large application, this can happen
quite often.
</div>

If not used correctly, preload can harm performance by making unnecessary
requests for resources that aren't used. In this application, `details.css` is
another CSS file located at the root of the project but is used for a separate
`/details route`. To show an example of how preload can be used incorrectly, add a
preload hint for this resource as well.

<pre class="prettyprint">
&lt;head&gt;
  &lt;!-- ... --&gt;
  &lt;link rel=&quot;preload&quot; href=&quot;main.css&quot; as=&quot;style&quot;&gt;
  <strong>&lt;link rel=&quot;preload&quot; href=&quot;details.css&quot; as=&quot;style&quot;&gt;</strong>
&lt;/head&gt;
</pre>

Reload the application and take a look at the **Network** panel. 
A request is made to retrieve `details.css` even though it is not being used by the web page. 

![Network panel with unecessary preload](./network-panel-three.png)

Chrome displays a warning in the **Console** panel when a preloaded resource is
not used by the page within a few seconds after it has loaded.

![Preload warning in console](./console.png)

Use this warning as an indicator to identify if you have any preloaded resources
that are not being used immediately by your web page. You can now remove the
unnecessary preload link for this page.

<pre class="prettyprint">
&lt;head&gt;
  &lt;!-- ... --&gt;
  &lt;link rel=&quot;preload&quot; href=&quot;main.css&quot; as=&quot;style&quot;&gt;
  <s>&lt;link rel=&quot;preload&quot; href=&quot;details.css&quot; as=&quot;style&quot;&gt;</s>
&lt;/head&gt;
</pre>

Stylesheet files aren't the only type of asset that can be preloaded. In this
application, a web font (`K2D`) is defined using the
[@font-face](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/webfont-optimization#defining_a_font_family_with_font-face)
rule in `main.css` and is the last resource fetched by the browser. Since this
font file is used for the page header, add a preload tag to fetch it even
sooner.

<pre class="prettyprint">
&lt;head&gt;
  &lt;!-- ... --&gt;
  &lt;link rel=&quot;preload&quot; href=&quot;main.css&quot; as=&quot;style&quot;&gt;
  <strong>&lt;link rel=&quot;preload&quot; href=&quot;fonts/K2D.woff2&quot; as=&quot;font&quot; crossorigin&gt;</strong>
&lt;/head&gt;
</pre>

For this resource, `font` is used as a value for the `as` attribute instead. 
Reloading the application will show that the font is now fetched earlier.

![Network panel after preloading font](./network-panel-four.png)

For a list of all the types of resources that can be fetched along with the
correct values that should be used for the `as` attribute, refer to the
[MDN article on Preloading](https://developer.mozilla.org/en-US/docs/Web/HTML/Preloading_content#What_types_of_content_can_be_preloaded).

<div class="aside note">
Cross-origin resources can also be preloaded using the <code>crossorigin</code> attribute. Moreover, same-origin font resources 
must be fetched using anonymous mode CORS which is why the <code>crossorigin</code> attribute is also used in this preload tag.

The <a href="/secure/cross-origin-resource-sharing">Cross Origin Resource Sharing</a> guide explains the topic of same-origin and cross-origin requests in more detail.
</div>

## Prefetch future resources

**Prefetch** is another browser hint that can be used to make 
a request for an asset used for a different navigation route but at 
a lower priority than other important assets needed for the current page.

In this website, clicking the image will take you to a separate `details/`
route.

![Details route](./details.png)

A separate CSS file, `details.css`, contains all the styles needed for this
simple page. Add a link element to `index.html` to prefetch this resource.

<pre class="prettyprint">
&lt;head&gt;
  &lt;!-- ... --&gt;
  <strong>&lt;link rel=&quot;prefetch&quot; href=&quot;details.css&quot;&gt;</strong>
&lt;/head&gt;
</pre>

To understand how this triggers a request for the file, open the **Network** panel in DevTools
and uncheck the **Disable cache** option.

![Disable cache in Chrome DevTools](./disable-cache.png)

Reload the application and notice how a very low priority request is made for
`details.css` after all the other files have been fetched.

![Network panel with prefetched resource](./network-panel-five.png)

With DevTools open, click the image on the website to navigate to the `details` page. 
Since a link element is used in `details.html` to fetch `details.css`, a request is made for the
resource as expected.

![Details page network requests](./network-panel-six.png)

Click the `details.css` network request in DevTools to view its details. You'll notice
that the file is retrieved from the browsers's disk cache.

![Details request fetched from disk cache](./details-css.png)

By taking advantage of browser idle time, prefetch makes an early request for a
resource needed for a different page. This speeds up future navigation requests
by allowing the browser to cache the asset sooner and serve it from the cache
when needed.

## Preloading and prefetching with webpack

The
[code splitting guide](/fast/reduce-javascript-payloads-with-code-splitting)
explored the use of dynamic imports to split a bundle into multiple chunks. To
refresh your memory, this was demonstrated with a simple application that 
dynamically imports a module from [Lodash](https://lodash.com/) when a form is submitted.

![Magic Sorter app that demonstrates code splitting](./magic.gif)

You can access [the Glitch for this application here](https://glitch.com/edit/#!/code-splitting).

The following block of code, which lives in `src/index.js,` is responsible for
dynamically importing the method when the button is clicked.

```
form.addEventListener("submit", e => {
  e.preventDefault()
  import('lodash.sortby')
    .then(module => module.default)
    .then(sortInput())
    .catch(err => { alert(err) });
});
```

Splitting a bundle will improve page loading times by
reducing its initial size. Version 4.6.0 of webpack provides support to preload or
prefetch chunks that are imported dynamically. Using this application as an
example, the `lodash` method can be prefetched at browser idle time; when a user
presses the button, there is no delay for the resource to be fetched.

Use the specific `webpackPrefetch` comment parameter within a dynamic import to prefetch a particular chunk.
Here is how it would look with this particular application.

<pre class="prettyprint">
form.addEventListener("submit", e => {
  e.preventDefault()
  <strong>import(/* webpackPrefetch: true */ 'lodash.sortby')</strong>
    .then(module => module.default)
    .then(sortInput())
    .catch(err => { alert(err) });
});
</pre>

Once the application is reloaded, webpack will inject a prefetch tag for the
resource into the head of the document. This can be seen in the **Elements**
panel in DevTools.

![Elements panel with prefetch tag](./devtools-elements.png)

Observing the requests in the **Network** panel will also show that this chunk is
fetched with a low priority after all other resources have been requested.

![Network panel with prefetched request](./another-network-panel.png)

Although prefetch makes more sense for this use case, webpack also provides support for preloading
chunks that are dynamically imported.

```
import(/* webpackPreload: true */ 'module')
```

## Conclusion

With this codelab, you should have a solid understanding of how preloading or prefetching certain assets can improve the user experience of your site. It is important to mention that these techniques should not be used for every resource and using them incorrectly can harm performance. The best results will be noticed by only preloading or prefetching selectively. 

To summarize:

+ Use **preload** for resources that are discovered late but are critical to the current page.
+ Use **prefetch** for resources that will be needed for a future navigation route or user action.

Not all browsers currently support both preload and prefetch. This means that
not all users of your application may notice performance improvements.

+ [Browser support - Preload](https://caniuse.com/#feat=link-rel-preload)
+ [Browser support - Prefetch](https://caniuse.com/#feat=link-rel-prefetch)

If you would like more information about specific aspects of how preloading and
prefetching can affect your web page, refer to these articles:

+ [Preload, Prefetch and Priorities in Chrome](https://medium.com/reloading/preload-prefetch-and-priorities-in-chrome-776165961bbf)
+ [&lt;link rel=&quot;prefetch/preload&quot;&gt; in webpack](https://medium.com/webpack/link-rel-prefetch-preload-in-webpack-51a52358f84c)