@page can-views Views
@parent api 2
@package ../../../package.json
@templateRender <% %>
@description Libraries for creating custom elements and rendering live-bound templates into HTML.

@body

CanJS views are [can-stache] templates, that implement a syntax similar to
[Mustache](https://mustache.github.io/mustache.5.html) and [Handlebars](http://handlebarsjs.com/),
and include special features like event bindings, custom elements, and performance optimizations.

[can-stache] templates look like HTML, but with _magic_ tags like [can-stache.tags.escaped]
and view bindings like [can-stache-bindings.twoWay] in the template. For example, the following is the application template in the [guides/todomvc]:

```html
<header id="header">
	<h1>todos</h1>
	<todo-create/>
</header>

<ul id="todo-list">
	{{#each todos}}
		<li class="todo {{#if complete}}completed{{/if}}">
				<div class="view">
						<input class="toggle" type="checkbox" checked:bind="complete">
						<label>{{name}}</label>
						<button class="destroy" on:click="destroy()"></button>
				</div>

				<input class="edit" type="text" value="{{name}}"/>
		</li>
	{{/each}}
</ul>
```

The following sections cover:

 - [The powerful syntaxes](#MustacheandHandlebarsextendedsyntax) that support the transformation of any ViewModel into HTML.
 - How [custom elements and attributes](#Customelementsandattributes) make
   applications easer to assemble and debug.
 - The [binding syntaxes](#DataandEventBindings) that allow HTML to
   call methods back on the ViewModel.
 - The strategies used to keep [DOM updates to a minimum](#MinimalDOMupdates).
 - How Views can fit in your larger ecosystem with [template minification](#Templateminification),
   [dependency declarations](#In_templatedependencydeclarations), and [progressive loading](#ProgressiveLoading).

### Mustache and Handlebars extended syntax

[can-stache] templates implement the mustache syntax
[Mustache](https://mustache.github.io/mustache.5.html), adopt many of
the [Handlebars](http://handlebarsjs.com/) extensions, and provide a few extensions of
their own.  The result is a simple syntax that covers the most common things needed in a template,
but is capable of translating any ViewModel into HTML.


[can-stache] is built on the Mustache spec because Mustache simplifies the most common needs of templates into
a very limited subset of syntax.  Most of Mustache is just:

 - [can-stache.tags.escaped] - to insert content into the page.
 - [can-stache.tags.section]...[can-stache.tags.close] - to do conditionals, looping, or change context.

A simple template might look like:

```html
<p>Hello {{name}}</p>
<p>You have just won {{value}} dollars!</p>
{{#in_ca}}
<p>Well, {{taxed.ca.value}} dollars, after taxes.</p>
{{/in_ca}}
```

This is not enough to translate every ViewModel into HTML, so [can-stache] supports
Handlebars helpers like [can-stache.helpers.each] and
the ability to [can-stache/expressions/call call methods].

A template that uses those features looks like:

```html
{{#players}}
    <h2>{{name}}</h2>
    {{#each stats.forPlayerId(id) }}
		<span>
			{{type}}
		</span>
	{{/each}}
{{/players}}
```





### Custom elements and attributes

CanJS supports defining custom elements and
attributes.  You can make it so adding a `<ui-datepicker>`
element to the page creates a date picker widget; or, you can make it so
a `my-tooltip="your message"` attribute adds a tooltip.

Custom elements are created for widgets like `<ui-datepicker>` and for
higher-order components like `<acme-message-editor>`.  Higher-order components
often assemble the behavior of multiple widget components.  Custom elements are created with [can-view-callbacks.tag] or [can-component].

Custom attributes are typically used for mixins that can be
added to any element. Custom attributes are created with [can-view-callbacks.attr].

The main advantages of building applications based on custom HTML elements and attributes are:

1. Ease of use - Designers can do it! Non-developers can express complex behavior with little to no JavaScript required. All you need to build a new page or feature is HTML.
2. Application assembly clarity - Applications assembled with custom elements are easier to debug and
   and understand the relationship between the user interface and the code powering it.

Let’s explore these benefits more in the following sections:

__Ease of use__

Before custom HTML elements existed, to add a date picker to your page, you would:

1. Add a placeholder HTML element

   ```html
   <div class='datepicker' />
   ```

2. Add JavaScript code to instantiate your datepicker:

   ```js
   $('.datepicker').datepicker(task.dueDate)
   ```

3. Wire up the datepicker to update the rest of your application and vice-versa:

   ```js
   task.on("dueDate", function(ev, dueDate){
       $('.datepicker').datepicker(dueDate)
   })

   $('.datepicker').on("datechange", function(ev, dueDate){
       task.dueDate = dueDate;
   });
   ```



With custom HTML elements, to add the same datepicker, you would
simply add the datepicker to your HTML or template:

```html
<ui-datepicker value:bind="task.dueDate"/>
```

That might seem like a subtle difference, but it is actually a major step forward. The custom HTML element syntax allows for instantiation, configuration, and location, all happening at the same time.

Custom HTML elements are one aspect of [Web Components](https://www.webcomponents.org/), a collection of browser specs that have [yet to be implemented](https://caniuse.com/#search=components) across browsers.

__Application assembly clarity__

Custom elements make it easier to tell how an application was assembled. This is because you
can inspect the DOM and see the custom elements and their bindings.  

The
following shows inspecting the [guides/todomvc]’s _“What needs to be done?”_ input element.  Notice how it’s easy to tell that its behavior is provided by the
`<todo-create>` element.


<img src="../../docs/can-guides/images/introduction/inspect.png" style="width:100%;max-width:750px" />


### Data and Event Bindings

[can-stache] includes Mustache data bindings that update the DOM when data changes.  For example,
if the data passed to the following template changes, the DOM is automatically updated.

```html
<h1 class="{{#if user.admin}}admin{{/if}}">Hello {{user.name}}</h1>
```

In addition to the default Mustache data bindings, the [can-stache-bindings] module
adds more powerful data and event bindings. These event bindings provide full control over how
data and control flows between the DOM, ViewModels, and the [can-view-scope]. Bindings look like:

 - `on:event="key()"` for [can-stache-bindings.event event binding].
 - `prop:from="key"` for [can-stache-bindings.toChild one-way binding to a child].
 - `prop:to="key"` for [can-stache-bindings.toParent one-way binding to a parent].
 - `prop:bind="key"` for [can-stache-bindings.twoWay two-way binding].

To two-way bind an `<input>` element’s `value` to a `todo.name` looks like:

```html
<input value:bind="todo.name"/>
```

To two-way bind a custom `<ui-datepicker>`’s `date` to a `todo.dueDate` looks like:

```html
<ui-datepicker date:bind="todo.dueDate"/>
```

By mixing and matching `$` and the different syntaxes, you have complete control over how
data flows.

### Minimal DOM updates

Everyone knows that updating the DOM is traditionally the slowest part of JavaScript
applications.  CanJS uses two strategies for keeping DOM updates to a minimum:
observation and data diffing.

To understand how these strategies are used, consider a template like:

```html
<ul>
{{#each completeTodos() }}
	<div>{{name}}</div>
{{/each}}
</ul>
```

And rendered with `viewModel` like:

```js
var ViewModel = DefineMap.extend({
    tasks: Todo.List,
    completeTodos: function(){
        return this.tasks.filter({complete: false});
    }
});

var viewModel = new ViewModel({
    tasks: new Todo.List([
        {name: "dishes", complete: true},
        {name: "lawn", complete: false}
    ])
})
```

__Observation__

CanJS directly observes what’s happening in each magic tag
like `{{name}}` so it can localize changes as much as possible. This means
that when the first todo’s name is changed like:

```js
viewModel.tasks[0].name = "Do the dishes"
```

This change will be observed, and a textNode in the div will simply
be updated with the new `name`.  There’s no diffing on the whole template.  A
change happens and we know directly what is impacted.

__Data diffing__

The [can-stache.helpers.each {{#each}} helper] provides data diffing.  It is able
to do a difference between two arrays and calculate a minimal set of mutations to
make one array match another.  This means that if a new task is added to the
list of `tasks` like:

```js
viewModel.tasks.push({name: "Understand diffing", complete: true})
```

This change will be observed, and a new array will be returned from
`completeTodos()`.  The `#each` helper will [can-util/js/diff/diff] this new array to the
original array, and only create a single new `<div>` for the new todo.  


### Template minification

While templates provide obvious benefits to application maintainability, they can be a
chore to correctly integrate into the build tool chain. The [steal-stache] library provides an easy hook to load [can-stache] templates using [StealJS](https://stealjs.com) and include the compiled templates into the minified result of the build.

[steal-stache] returns a renderer function that will render the template into a document fragment.

```js
import todosStache from "todos.stache"
todosStache([{name: "dishes"}]) //-> <documentFragment>
```

When the build is run, this import statement will tell StealJS that "todos.stache" is a dependency, and will include an already parsed representation in the build result.

### In-template dependency declarations

[can-view-import](../../can-view-import.html) allows templates to import their dependencies like
other modules. You can import custom elements, helpers, and other modules straight from a template module like:

```html
<can-import from="components/my_tabs"/>
<can-import from="helpers/prettyDate"/>
<my-tabs>
  <my-panel title="{{prettyDate start}}">...</my-panel>
  <my-panel title="{{prettyDate end}}">...</my-panel>
</my-tabs>
```

### Progressive Loading

A template may load or conditionally load a module after the initial page load. `<can-import>` allows progressive loading by using an end tag.

This feature, when used with [steal-stache](../../steal-stache.html), signals to the build that the enclosed section’s dependencies should be dynamically loaded at runtime.

```html
{{#eq location 'home'}}
<can-import from="components/home">
  <my-home/>
</can-import>
{{/eq}}
{{#eq location 'away'}}
<can-import from="components/chat">
  <my-chat/>
</can-import>
{{/eq}}
```
