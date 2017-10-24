@page can-polyfills Polyfills
@parent api 8
@package ../../../package.json
@templateRender <% %>
@description Libraries that are polyfill-like by providing implementations of various JavaScript APIs.

@body

**[can-symbol]** <small><%can-symbol.package.version%></small> <%can-symbol.package.description%>

**[can-vdom]** <small><%can-vdom.package.version%></small> <%can-vdom.package.description%>

## Server Side Rendering

CanJS applications can be rendered on the server by running the same code. This is known as [Isomorphic JavaScript](http://isomorphic.net/javascript) or [Universal JavaScript](https://medium.com/@mjackson/universal-javascript-4761051b7ae9).

Server-side rendering (SSR) provides two main benefits over traditional single page apps: better page load performance and SEO support.

CanJS makes it possible to load your application on the server. This is because CanJS works in a NodeJS context, on top of a virtual DOM.

Using [can-vdom](../can-vdom.html) and [can-zone](../can-zone.html), you can set up your own server side rendering system, such as [the one used in DoneJS](https://donejs.com/Apis.html#server-side-rendering-apis). For information on using SSR without setting anything up yourself, please check out the DoneJS [quick start](https://donejs.com/Guide.html) and [in depth](https://donejs.com/place-my-order.html) guides.
