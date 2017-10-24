@page can-data-modeling Data Modeling
@parent api 1
@description Libraries for defining your data’s schema and communicating with a server to read and write data.

@body

In addition to CRUD and real-time capabilities, [can-set] and [can-connect] provide lots of great features out-of-the-box:

- [Memory-safe instance store](#Memory_safeinstancestore)
- [Real-time list updates](#Real_timelistupdates)
- [Parameter awareness](#Parameterawareness)
- [Caching and minimal data requests](#Cachingandminimaldatarequests)
- [Related-data features](#Related_datafeatures)

We’ll cover each of these in the sections below.

### Separation of concerns

CanJS separates your model layer into two parts:

1) Communicating with a server.
2) Managing your data’s schema.

Separating these two concerns means your model data isn’t tied to how you communicate with your API. Your project may start with a RESTful API for CRUD operations but end up with a real-time WebSocket API, and with CanJS, that change doesn’t affect how your data is modeled.

Additionally, with our mixin-based approach, you can easily add behaviors to both parts separately. Want to add [can-connect/data/localstorage-cache/localstorage-cache Local Storage] caching? It’s a one-line add-on. How about a behavior to [can-connect/data/combine-requests/combine-requests efficiently combine network requests]? One line too! Need something not provided by [can-connect]? Write and mix in your own custom behaviors.

This separation of concerns and powerful mixin behavior is accomplished by encapsulating the code required to connect to a service and encouraging typed definitions of your model data. For every “type” of data object in your project, you can create a model to represent the properties and methods attached to it. With this model in hand, you can structure how you communicate with your server. Different API calls can return the same type of data and have those represented as the same model objects.

Let’s look at an example of how we would define a `Todo` type and a list of todos:

```js
var DefineList = require("can-define/list/list");
var DefineMap = require("can-define/map/map");

var Todo = DefineMap.extend({
	complete: "boolean",
	name: "string"
});

Todo.List = DefineList.extend({
	"#": Todo,
	completeCount: function(){
		return this.filter({complete: true}).length;
	}
})
```

This example uses [can-define/map/map] to create a type definition for a `Todo`; each instance of `Todo` has a boolean `complete` property and a string `name` property.

This example also uses [can-define/list/list] to define a type for an array of `Todo` instances; the list has a `completeCount` method for easily determining how many todos in the list have been completed.

Using [can-connect], we’ll create a connection between a RESTful `/api/todos` service and our `Todo` instances and `TodoList` lists:

```js
var connect = require("can-connect");
Todo.connection = connect([
	require("can-connect/can/map/map"),
	require("can-connect/constructor/constructor"),
	require("can-connect/data/url/url")
], {
	url: "/api/todos",
	Map: Todo,
	List: TodoList
});
```

That connection can be used to get a `Todo.List` of `Todo`s:

```js
Todo.getList({}).then(function(todos) {
	// Do what you’d like with the `todos`
});
```

### Memory-safe instance store

Let’s continue with our todo app example and imagine that we want to show two lists on a page: incomplete and urgent todos.

First, let’s fetch the incomplete todos:

```js
Todo.getList({completed: false}).then(function(incompleteTodos) {});
```

`incompleteTodos` might look like this:

    [
      {id: 2, completed: false, name: "Finish docs", priority: "high"},
      {id: 3, completed: false, name: "Publish release", priority: "medium"}
    ]

Next, let’s fetch a list of high-priority todos:

```js
Todo.getList({priority: "high"}).then(function(urgentTodos) {});
```

`urgentTodos` might look like this:

    [
      {id: 1, completed: true, name: "Finish code", priority: "high"},
      {id: 2, completed: false, name: "Finish docs", priority: "high"}
    ]

Note that the “Finish docs” todo appears in both lists. If we make a change to the todo (e.g. changing its name), we want that change to appear in both lists.

[can-connect]’s [can-connect/constructor/store/store.instanceStore instance store] keeps a reference to every model object by `id` (but you can [can-set.props.id change] which property is used). It does two things:

1. Prevents duplicate instances of a model object from being created.
2. Cleans up unused instances to release memory when they’re no longer referenced.

Let’s look at both of these points in more detail.

#### Duplicate instances

The instance store prevents duplicate instances from being created by storing each model object by its [can-set.props.id id]. When a model object is fetched from the server, CanJS checks its `id` to see if it’s already in the instance store; if it is, then CanJS will reuse the same object.

In our example, CanJS puts the “Finish docs” todo in the instance store when `incompleteTodos` is fetched. When `urgentTodos` is retrieved, CanJS sees the “Finish docs” todo with the same `id`, so it reuses the instance of “Finish docs” that is already in the instance store.

If these todos are displayed in separate lists on the page, and a user marks “Finish docs” as completed in one of the lists (causing the `completed` property to be set to `true`), then the other list will reflect this change.

#### Prevent memory leaks

A global instance store _sounds_ great until you consider the memory implications: if every model object instance is tracked, then won’t the application’s memory usage only grow over time?

CanJS solves this potential problem by keeping track of which objects are observing changes to your model object instances.

The reference count for each object increases in two ways:

- __Explicitly:__ if you use [can-connect/constructor/store/store.addInstanceReference] or call `.on()` on an instance (e.g. `todo.on('name', function(){})`)

- __Implicitly:__ if properties of the instance are bound to via live-binding in a view, e.g. `Name: {{name}}` in a [can-stache] template

Similarly, the reference count is decreased in two ways:

- __Explicitly:__ if you use [can-connect/constructor/store/store.deleteInstanceReference] or call `.off()` on an instance

- __Implicitly:__ if part of the DOM connected to a live-binding gets removed

When the reference count for a model object instance gets back down to 0 (no more references), the instance is removed from the store so its memory can be garbage collected.

The result is that in long-running applications that stream large amounts of data, this store will not cause memory to increase unnecessarily over time.

You can read more about the benefits of the instance store in our [“Avoid the Zombie Apocalypse” blog post](https://www.bitovi.com/blog/avoid-the-zombie-apocalypse).

### Real-time list updates

CanJS also automatically inserts, removes, and replaces objects within lists.

Let’s continue with our incomplete and urgent todo example from the previous section.

`incompleteTodos` looks like this:

    [
      {id: 2, completed: false, name: "Finish docs", priority: "high"},
      {id: 3, completed: false, name: "Publish release", priority: "medium"}
    ]

`urgentTodos` looks like this:

    [
      {id: 1, completed: true, name: "Finish code", priority: "high"},
      {id: 2, completed: false, name: "Finish docs", priority: "high"}
    ]

In the UI, there’s a checkbox next to each urgent todo that toggles the `completed` property:

```js
todo.completed = !todo.completed;
todo.save();
```

When the user clicks the checkbox for the “Finish docs” todo, its `completed` property is set to `true` and it disappears from the `incompleteTodos` list when [can-connect/can/map/map.prototype.save .save()] is called.

This is made possible by two things:

- The [can-connect/constructor/store/store.listStore list store] contains all of the lists loaded from the server. It’s memory safe so it won’t leak.

- [can-set] understands what your parameters mean so it can insert, remove, and replace objects within your lists. This is discussed in the following _"Parameter awareness"_ section.

CanJS’s real-time list updates work great with "push notification" systems like [socket.io](https://socket.io/) and SignalR.  To add realtime behavior to a CanJS app, you
just have to call the [can-connect/real-time/real-time.createInstance],
[can-connect/real-time/real-time.updateInstance] and [can-connect/real-time/real-time.destroyInstance]
when updates happen similar to the following:

```js
var socket = io('https://example.com');

socket.on('todo created', function(todo){
    Todo.connection.createInstance(todo)
});
socket.on('todo updated', function(todo){
    Todo.connection.updateInstance(todo)
});
socket.on('todo removed', function(todo){
    Todo.connection.destroyInstance(todo)
});
```

### Parameter awareness

When you make a request for `incompleteTodos` like the one below:

```js
Todo.getList({completed: false}).then(function(incompleteTodos) {});
```

The `{completed: false}` object is passed to the server as parameters and represents all incomplete todos. You can configure a connection with [can-set.Algebra] that understands these parameters.

Here’s an example of [can-connect/base/base.algebra setting up the algebra] for the `Todo.connection`:

```js
var connect = require("can-connect");
var set = require("can-set");

Todo.algebra = new set.Algebra(
	set.props.boolean("completed")
);

Todo.connection = connect([
	require("can-connect/can/map/map"),
	require("can-connect/constructor/constructor"),
	require("can-connect/data/url/url")
], {
	url: "/api/todos",
	Map: Todo,
	List: Todo.List,
	algebra: Todo.algebra
});
```
@highlight 4-6,16-16

The `{completed: false}` parameters are associated with `incompleteTodos` so `can-connect` knows that `incompleteTodos` should contain _any_ todo with a `false` `completed` property. By understanding what
the parameters used to request data mean, all sorts of interesting behaviors and performance optimizations
can happen, including:

 - Real-time updates as described in the previous section.
 - Fall-through caching, request caching, and combining requests behaviors as described in the
 following sections.

Parameter awareness is provided by [can-set].  Read more about the magic of `can-set` in its [can-set API docs].

### Caching and minimal data requests

Undoubtedly, the slowest part of any web application is communicating with the server. CanJS uses the following strategies to improve performance:

* [can-connect/fall-through-cache/fall-through-cache Fall-through caching]: improve perceived performance by showing cached data first (while still fetching the latest data)
* [can-connect/cache-requests/cache-requests Request caching]: reduce the number and size of server requests by intelligently using cached datasets
* [can-connect/data/combine-requests/combine-requests Combining requests]: combine multiple requests to the same API into one request

#### Fall-through caching

To increase perceived performance, `can-connect` includes a [can-connect/fall-through-cache/fall-through-cache fall-through cache] that first serves cached data from `localStorage` while simultaneously making the API request to get the latest data.

The major benefit of this technique is improved perceived performance: users will see content faster because it’s returned immediately from the cache. When the data hasn’t changed, the user doesn’t notice anything, but when it has, the magic of live-bindings automatically updates the data as soon as the API request finishes.

#### Request caching

In some scenarios, an even more aggressive caching strategy is favorable. One example is fetching data that doesn’t change very often, or cached data that you can invalidate yourself. The [can-connect/cache-requests/cache-requests] behavior can reduce both the number of requests that are made and the size of those requests in these cases.

In the first scenario, where the data doesn’t change very often (and thus shouldn’t be fetched again during the lifetime of the application), no more requests to the API will be made for that same set of data. In the second scenario, you can choose to invalidate the cache yourself, so after the first API request the data is always cached until you clear it manually.

Additionally, the request logic is more aggressive in its attempts to find subsets of the data within the cache and to only make an API request for the subset NOT found in the cache. In other words, partial cache hits are supported.

#### Combining requests

CanJS collects requests that are made within [can-connect/data/combine-requests.time a millisecond] of each other and tries to combine them into a single request if they are for the same API.

For example, let’s say we’re loading a page that has two parts: a section with incomplete todos and a section that’s an archive of completed todos. The incomplete section is just a list of todos, while the archive section is broken up by month, so you want to split these sections into two different components.

In most other frameworks, you would probably decide to have some parent component fetch the list of all todos so you could pass different subsets to each component. This decreases the reusability and maintainability of the components, but it would result in just one network request instead of two.

With CanJS, you don’t have to choose between maintainability and performance. You can decide to have each component fetch its data independently and [can-connect] will intelligently combine the two requests into one.

This is made possible by the [can-set] algebra we discussed earlier. [can-connect] sees the outgoing requests, can determine that requests for `Todo.getList({completed: true, sort: 'completedDate'})` and `Todo.getList({completed: false, sort: 'priority'})` are equivalent to just one `Todo.getList({})` request, then make that single request and return the correct sorted data to each call.

This [can-connect/data/combine-requests/combine-requests configurable behavior] is extremely powerful because it abstracts network request complexity away from how you create and compose your application.

### Related-data features

CanJS makes dealing with document-based APIs easier by handling situations where the server might return either a reference to a value or the value itself.

For example, in a MongoDB setup, a request like `GET /api/todos/2` might return:

```
{
  id: 2,
  name: "Finish docs",
  projectRef: 1
}
```

But a request like `GET /api/todos/2?$populate=projectRef` might return:

```
{
  id: 2,
  name: "Finish docs",
  projectRef: {
	id: 1,
	name: "Release"
  }
}
```

[can-connect/can/ref/ref] handles this ambiguity by creating a [can-connect/can/ref/ref.Map.Ref Ref type] that is always populated by the `id` and can contain the full value if it’s been fetched.

For example, without populating the project data:

```js
Todo.get({id: 2}).then(function(todo){
  todo.projectRef.id //-> 2
});
```

With populating the project data:

```js
Todo.get({id: 2, populate: "projectRef"}).then(function(todo){
  todo.projectRef.id //-> 2
});
```

The values of other properties and methods on the [can-connect/can/ref/ref.Map.Ref Ref type] are determined by whether the reference was populated or the referenced item already exists in the [can-connect/constructor/store/store.instanceStore].

For example, `value`, which points to the referenced instance, will be populated if the reference was populated:

```js
Todo.get({id: 2, populate: "projectRef"}).then(function(todo){
  todo.projectRef.value.name //-> “Release”
});
```

Or, it can be lazy loaded if it’s used in a template. For example, with this template:

```html
{{#each todos as todo}}
  Name: {{todo.name}}
  Project: {{todo.projectRef.value.name}}
{{/each}}
```

If `todo.projectRef.value` hasn’t been loaded by some other means, CanJS will fetch it from the server so it can be displayed in the template. This is handled automatically without you having to write any additional code to fetch the project data.

Additionally, if multiple todos have the same project, only one request will be made to the server (if the data isn’t already cached), thanks to the [can-connect/data/combine-requests/combine-requests] behavior.
