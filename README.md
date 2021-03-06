## DXF: An XSS filter for DOM-Based XSS Attacks

DXF is a fast DOM-Based XSS filter that runs entirely in JavaScript using ES6
features. Unlike other filters, it does not require modifications to the
JavaScript engine. It also does not perform intrusive source-to-source
transformations that increase page loading time or runtime overhead.

DXF currently only runs in Firefox, but it uses standard ES6 features
available in Chrome and IE >= 10.

DXF monitors all JavaScript sources (e.g. `document.location`) and sinks (e.g.
`document.write`). Before sinks are executed, taint-inteference is used to
infer a flow between any of the sources and that particular sink. If the flow
generates new JavaScript code, the filter prevents the sink from executing.
Sources are monitored because, although we cannot track flows explicitly, we
can immediately exclude any sources that have not been touched from the
matching process. In this sense, source monitoring is more of an optimization
than a requirement.

Although the filter can be implemented as an HTTP proxy, we implement it here
as a Firefox extension using the add-on SDK.

# Installation and Usage

Install the following:

- emscripten
- node >=5.1.1
- babel
- uglifyjs

Then run `make xpi` and drag&drop the XPI into your FF42 installation.

# Implementation

The filter is made of the following components:

* *Rewriter* (`extension/rewriter.js`): implements code rewriting,
to rename all references to objects that lead to sources and sinks which
cannot be otherwise monitored at runtime (e.g. non-configurable properties
that cannot be wrapped, like `window`, `document`, `location`, or non-
wrappeable property like `eval` because of direct eval semantics). The runtime
will then be able to monitor the rewritten reference (e.g. by using a proxy on
`__window`).

* *Proxy* (`extension/index.js / addon-proxy`): uses the rewriter on all
incoming HTML and JS files. Additionally, it exports some functionality from
the proxy extension to web content. This functionality could also be exported
separately to web content with a decent build system, but that's how it is now.

* *Runtime Monitoring* (`setup.js / utils.js`): these scripts execute before
any other scripts on the page. They wrap all configurable sources and sinks,
and define proxies for the non-configurable ones, which the rewriter has
redirected (e.g. `window` to `__window`). The purpose of this wrapping is to
intercept sinks and check for XSS attacks.

* *Taint-Inference Engine* (`p1.js / p1utils.js`): `p1.js` is an asm.js module
based on `nlearn`, an approximate substring matching routine used in
XSSFilt. ``p1utils.js` exports the high-level function `p1FastMatch`.

## Wrapping Sources and Sinks

Here we detail the main sources and sinks and explain how we interpose on them
(roughly based on the DOMXSSWiki).

Sources:

- `document.(URL|documentURI)`: equivalent to the URL of the page, which is
always relevant for DOM-Based XSS, so it's always included. Wrapped as a
configurable property of `Document.prototype`

- `document.baseURI`: wrappable as a configurable property of `Node.prototype`
(every node has a baseURI), but is it really a legitimate sink? If you can
modify the base URL through reflected XSS, you have bigger fishes to fry.

- `location(.(href|search|hash|pathname))?`: `location` is a non-configurable
property of `document` and `window`. The only way to interpose on it is to
rewrite `document`, `window` and `location` itself to use proxies. *Getting*
`location.href` is the same as `document.URL`, which is always included.

- `document.cookie`: wrappable, configurable prop of `HTMLDocument.prototype`.

- `document.referrer`: wrappable, configurable prop of `Document.prototype`.

- `window.name`: wrappable, configurable prop of `window`.

- `history`: wrappable, configurable prop of `window`. It can affect
`location.href`, but do we care about tracking the value of that over time?
TODO

- `localStorage`: wrappable, configurable. TODO

- `sessionStorage`: wrappable, configurable. TODO

- `onmessage - event.data`: configurable on `MessageEvent.prototype`, but i am
wondering if this is only necessary in limited cases. Same-origin messages can
be excluded for example. TODO


Sinks:

- `eval`: configurable on `window`, but it cannot be wrapped without breaking
direct eval semantics. direct evals will use `__peval` and indirect evals will
use `__indirectEval`, and `__window.eval` will redirect to `__inidirectEval`.

- `Function`: wrappeable, configurable. 

- `setTimeout/setInterval`: wrappable, configurable.

- `scriptEl.src`: wrappeable, configurable on `HTMLScriptElement.prototype`.

- `scriptEl.textContent`: wrappeable, configurable on `Node.prototype`.

- `scriptEl.innerText`: not a sink in FF.

- `location(.(href|search|hash|pathname))?`: `location` is non-configurable on
`window` and `document`, and each one of its members are non-configurable.
Requires rewriting and proxying of `window`, `document`, `location`. Only `href` and
`search` are valid vectors for now, because they need to inject "javascript:" at the
beginning.

- `document.(write|writeln)`: wrappeable, configurable on `HTMLDocument.prototype`.

- `el.innerHTML`: wrappeable, configurable on `Element.prototype`.

- `Range`: can create document fragments, but they still need to be appended.
Probably does not need to be addressed directly. NOT NECESSARY?

Other problems:

- `parent/top/frames[i].obj`: `parent` and `frames` are wrapped, top is rewritten as `__top`.

- `self`: wrappeable, configurable on `window`.

- `iframeEl.contentWindow`: direct access to other frames' `window`. Not only
access needs to be redirected to `__window`, but in some cases the environment
itself must be initialized (e.g. `about:blank`). Wrappeable, configurable on
`HTMLIFrameElement.prototype`.

- `window.window`: direct access to `window`. non-configurable, __window must
intercept it.
