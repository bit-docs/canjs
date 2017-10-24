@page can-observables Observables
@parent api 0
@description Libraries for creating observable objects and values.

@body

## Computes

CanJS has two powerful observable systems that are the foundation for many of the other
core libraries:

 - [can-compute] - Observable values and derived observable values.
 - [can-define] - Observable objects.

This section is about the technical highlights of [can-compute].  However,
as [can-define] uses computes internally for [computed getter properties](#Computedgetterproperties)
and [asynchronous computed getter properties](##Asynccomputedgetterproperties), the benefits
of computes extend to [can-define]. In a few examples cases, we’ll use [computed getter properties](#Computedgetterproperties) to
show the advantages of computes.

[can-compute] is used in similar situations as event streams libraries like RXJS and Bacon.js. Computes
are used to transform a set of observable values into another observable value.  While event stream libraries are able to set up more complex transformations, computes can set up simple but common transformations more easily.

For example, the following compute keeps the completed count of todos in a list:

```js
var DefineList = require("can-define/list/list");
var DefineMap = require("can-define/map/map");
var compute = require("can-compute");

var todoList = new DefineList([
    {name: "dishes",  complete: true},
    {name: "laundry", complete: false}
]);

var completedCount = compute(function(){
    return todoList.filter(function(todo){
        return todo.complete;
    });
})
```

`completedCount` is updated when any todo’s `complete` property changes like:

```js
todoList[0].complete = false;
```

Or a todo is added to or removed from the list like:

```js
todoList.push({name: "learn about computes", complete: true})
```

`completedCount` automatically listens to all of these changes because
[can-compute] infers dependencies.  Computes also:

 - [Cache their value](#Cachedvalues) for faster reads.
 - [Dispatch events synchronously](#Synchronous) for easier testing and debugging.
 - [Update only once for batched changes](#Batchedevents) for faster updates.

### Inferred dependencies

In event stream libraries or other computed libraries, you must declare your
dependencies like:

```js
var fullNameStream = Kefir.combine(firstNameStream, lastNameStream, function(firstName, lastName){
    return firstName + " " + lastName;
});
```

or like:

```js
fullName: Ember.computed('firstName', 'lastName', function() {
	return this.get('firstName')+" "+this.get('lastName');
});
```

[can-compute] infers its own dependencies without needing to explicitly declare them, therefore requiring less boilerplate code. This means you can write `fullName` like:

```js
var fullName = compute(function(){
    return firstName() + " " + lastName();
});
```

or like:

```js
Person = DefineMap.extend({
    firstName: "string",
    lastName: "string",
    get fullName() {
        return this.firstName + " " +this.lastName;
    }
});
```

This ability is especially useful when the dependencies are dynamic as in the
following `completedCount` example:


```js
var todoList = new DefineList([
    {name: "dishes",  complete: true},
    {name: "laundry", complete: false}
]);

var completedCount = compute(function(){
    return todoList.filter(function(todo){
        return todo.complete;
    });
})
```

When todos are added to and removed from `todoList`, `completedCount`
will update its bindings automatically.


### Cached values

Once a compute is bound (using [can-compute.computed.on] or [can-compute.computed.addEventListener]), it immediately calculates its
value and caches it so any future reads will use the cached value.

In the following example, before `fullName` is bound,
`fullName`’s value is recalculated every time it is read.  After `fullName` is bound,
its value is recalculated only when a dependent value changes.

```js
var compute = require("can-compute");
var firstName = compute("Payal");
var lastName = compute("Meyer");

var fullName = compute(function(){
    console.log("Calculating fullName.");
    return firstName()+" "+lastName();
});

fullName() // console.logs "Calculating fullName."
           //-> "Payal Meyer"

fullName() // console.logs "Calculating fullName."
           //-> "Payal Meyer"

fullName.on("change", function(){}) // console.logs "Calculating fullName."

fullName() //-> "Payal Meyer"
fullName() //-> "Payal Meyer"

firstName("Ramiya") // console.logs "Calculating fullName."

fullName() //-> "Ramiya Meyer"
```

Using cached values improves performance in situations where a computed value is frequently read by multiple parts of the application.  

### Synchronous

CanJS observables synchronously notify any event listeners. This makes testing
and debugging quite easier.

The following example shows how you can
change the `firstName` value and immediately check the consequences of that change:

```js
var stache = require("can-stache");
var compute = require("can-compute");

var template = stache("<h1>Welcome {{fullName}}</h1>");

var firstName = compute("Justin");
var lastName = compute("Meyer");

var fullName = compute(function(){
    return firstName()+" "+lastName();
});

var frag = template({fullName: fullName});

assert.equal(frag.firstChild.innerHTML, "Welcome Payal Meyer");

firstName("Ramiya");

assert.equal(frag.firstChild.innerHTML, "Welcome Ramiya Meyer");
```

### Batched events

The previous section highlighted that synchronous event
[can-event/batch/batch.dispatch dispatching] and DOM updates are ideal for many scenarios. But, there are times where this can cause performance problems. To prevent unnecessary updates, events can be batched using [can-event/batch/batch.start batch.start] and [can-event/batch/batch.stop batch.stop]. Computes and the DOM will only be updated once for all changes within the batch.

In the previous example, `{{fullName}}` would be updated twice
if `firstName` and `lastName` are changed:

```js
firstName("Payal");
lastName("Shah");
```

Wrapping this in a batch makes `{{fullName}}` update only once:


```js
var batch = require("can-event/batch/batch");

batch.start();
firstName("Payal");
lastName("Shah");
batch.stop();
```

Using [can-event/batch/batch.start batch.start] and [can-event/batch/batch.stop batch.stop]
can even make quadratic updates (`O(n^2)`) become linear (`O(n)`).

Consider the performance of a `completeAll` method that completes every todo in a list
and a `completeCount` compute that calculates the number of complete todos:

```js
var todoList = new DefineList([
    {name: "dishes",  complete: false},
    {name: "laundry", complete: false}
]);

var completeAll = function(){
    todoList.forEach(function(todo){
        console.log("completing", todo.name)
        todo.complete = true;
    });
};

var completedCount = compute(function(){
    return todoList.filter(function(todo){
        console.log("  checking", todo.name);
        return todo.complete;
    });
});

completedCount.on("change", function(ev, newVal){
    console.log("completedCount is", newVal);
});
```

If `completeAll` is called, the following will be logged:

```js
completeAll();
// console.logs "completing dishes"
// console.logs "  checking dishes"
// console.logs "  checking laundry"
// console.logs "completedCount is 1"
// console.logs "completing laundry"
// console.logs "  checking dishes"
// console.logs "  checking laundry"
// console.logs "completedCount is 2"
```

This means that every time a todo is marked as complete, `completedCount` loops
through every todo.

However, changing `completeAll` to use `batch.start` and `batch.stop` like:

```js
var completeAll = function(){
    batch.start();
    todoList.forEach(function(todo){
        console.log("completing", todo.name)
        todo.complete = true;
    });
    batch.stop()
};
```

means `completeAll` will log the following:

```js
completeAll();
// console.logs "completing dishes"
// console.logs "completing laundry"
// console.logs "  checking dishes"
// console.logs "  checking laundry"
// console.logs "completedCount is 2"
```

[can-event/batch/batch.start batch.start] and [can-event/batch/batch.stop batch.stop]
can improve performance by preventing compute recalculations.

## Observable Objects

[can-define] is used to create observable [Models](#MalleableModels) and [ViewModels](#VeraciousViewModels) like:

```js
var DefineMap = require("can-define/map/map");

var Person = DefineMap.extend({
    first: "string",
    last: "string",
    get fullName(){
        return this.first + " " + this.last;
    }
})
```

[can-define] uses [can-compute] internally to support [computed getter properties](##Computedgetterproperties) like the previous example’s `fullName`, so make sure to read about the benefits of [cool computes](#CoolComputes).

As [can-define] powers almost everything in a CanJS application, it has grown to be
quite powerful, performant and flexible.  Read on to explore some of its best characteristics.

### Expressive property definition syntax

[can-define] supports an expressive, powerful syntax for defining properties on observable objects and lists. It supports [can-define.types.get getter], [can-define.types.set setter],
initial [can-define.types.value], and [can-define.types.type] conversion, [can-define.types.serialize]
and [can-define-stream.stream] behaviors.

The following illustrates the signatures of these behaviors:

```js
DefineMap.extend({
    propertyName: {
        get: function(lastSetValue, resolve){ ... },
        set: function(newValue, resolve){ ... },
        type: function(newValue, propertyName){ ... },
        Type: Constructor,
        value: function(){ ... },
        Value: Constructor,
        serialize: function(){ ... },
        stream: function(setStream){ ... }
    }
})
```

[can-define] also supports a wide variety of short hands for setting up these
behaviors. The following illustrates some of these behaviors:

```js
DefineMap.extend({
    propertyA: Object      -> PropertyDefinition
    propertyB: String      -> {type: String}
    propertyC: Constructor -> {Type: Constructor}
    propertyD: [PropDefs]  -> {Type: DefineList.extend({"#": PropDefs})>}
    get propertyE(){...}   -> {get: propertyE(){...}}
    set propertyF(){...}   -> {get: propertyF(){...}}
    method: Function
})
```

Putting it together, the following defines an `Address` and `Person` type with some nifty features:

```js
// Address has a street, city, and state property
var Address = DefineMap.extend({
    street: "string",
    city: "string",
    state: "string"
})

var Person = DefineMap.extend({
    // first is a string
    first: {type: "string"},
    // last is a string
    last: "string",
    // fullName is the combination of first and last
    get fullName(){
        return this.first+" "+this.last;
    },
    // age is a number that defaults to `0`
    age: {value: 0, type: "number"},
    // addresses is a DefineList of Address types
    addresses: [Address]
});
```

### Object-oriented and functional

CanJS’s observables produce data types that are easy for others to consume,
but can be implemented with the rigor of declarative programming.  This is
accomplished by combining the benefits of object-oriented programming,
functional programming, and functional reactive programming.

[Functional programming](https://en.wikipedia.org/wiki/Functional_programming), which is a
form of [declarative programming](https://en.wikipedia.org/wiki/Declarative_programming), avoids
changing state and mutable data.  It treats programming as math.  This eliminates side effects,
making it easier to predict the behavior of an application.

> Programming is, at its best, a branch of formal mathematics and applied logic.   
> __Edsger Dijkstra__ - _1 March 1999 at the ACM Symposium on Applied Computing at San Antonio, TX_

However, [object-oriented](https://en.wikipedia.org/wiki/Object-oriented_programming) APIs often feel more natural.

> Object-oriented programming leverages the fact that humans have millions of years of evolution invested in conceiving of the world in terms of things, which have properties, and associated methods of doing things with them. A salt shaker has a property of the amount of salt in it, and can be shaken.  
> [Tim Boudreau, Oracle Labs](https://www.quora.com/Why-did-Dijkstra-say-that-%E2%80%9CObject-oriented-programming-is-an-exceptionally-bad-idea-which-could-only-have-originated-in-California-%E2%80%9D)

We agree with both of these ideas! The following object-oriented `SaltShaker` API feels intuitive - any
developer can immediately understand it.

```js
var saltShaker = new SaltShaker();

saltShaker.fill();  

saltShaker.shake() //-> "salt"
saltShaker.shake() //-> "salt"  
saltShaker.shake() //-> null   

saltShaker.empty   //-> true
```

To satisfy this API, `SaltShaker` could be implemented as follows:

```js
var DefineMap = require("can-define/map/map");

SaltShaker = DefineMap.extend({
    saltCount: {type: "number", value: 0},
    fill: function(){
        this.saltCount = 2;
    },
    shake: function(){
        var hasSalt = this.saltCount;
        this.saltCount = hasSalt ? this.saltCount - 1 : 0;
        return hasSalt ? "salt" : null;
    },
    get empty() {
        return ! this.saltCount;
    }
});
```

While `empty` is implemented [declaratively](https://en.wikipedia.org/wiki/Declarative_programming),
notice how both `fill` and `shake` mutate the state of `saltCount`.  In a more complex type,
this can easily lead to bugs.  Instead, the following uses [can-define-stream] and
[functional reactive programming](https://en.wikipedia.org/wiki/Functional_reactive_programming)
to make `saltCount` a function of the calls to `fill` and `shake`:

```js
var SaltShaker = DefineMap.extend({
    saltCount: {
        stream: function() {
            return this.stream("fill")
					.merge(this.stream("shake"))
					.scan(function(prev, event){
				if(event.type === "fill") {
					return 2;
				} else {
					return prev > 0 ? prev - 1 :  0;
				}
			},0);
        }
    },
    fill: function() {
        this.dispatch("fill");
    },
    shake: function() {
		var hadSalt = this.saltCount;
        this.dispatch("shake");
        return hadSalt ? "salt" : null;
    },
    get empty() {
        return !this.saltCount;
    }
});
```

CanJS provides three powerful functional helpers on [can-define/map/map] and [can-define/list/list] that will explore in the following sections:

 - [can-define.types.get computed getter properties]
 - [can-define.types.get async computed getter properties]
 - [can-define-stream.stream streamed properties]

### Computed getter properties

[can-define.types.get Computed getters] are the easiest way to declaratively transform
stateful values into derived values.  For example, the following defines a `completedCount`
property on instances of the `TodoList` type:

```js
var TodoList = DefineList.extend({
    "#": Todo,
    get completedCount(){
        return this.filter({complete: true}).length
    }
});

var todos = new TodoList([{complete: true}, {complete:false}]);
todos.completedCount //-> 1
```

These [can-define.types.get getters] are made with [can-compute], so they
[infer dependencies](#Inferreddependencies), [cache their values](#Cachedvalues), and are [synchronous](#Synchronous).

### Async computed getter properties

It’s common to load data asynchronously given some state. For example, given
a `todoId`, you might need to load a `todo` from the server.  This `todo` property
can be described using [can-define.types.get asynchronous computed getters] as follows:

```js
var EditTodoVM = DefineMap.extend({
    todoId: "number",
    todo: {
        get: function(lastSetValue, resolve){
            Todo.get(this.todoId).then(resolve);
        }
    }    
});
```

### Streamed properties

When the behavior of properties can’t be described with computes,
the [can-define-stream] module adds the ability to work with event
streams.  For example, `lastValidName` keeps track of `Person`’s
last `name` property that includes a space.

```js
var Person = DefineMap.extend({
  name: "string",
  lastValidName: {
    stream: function(){
      return this.stream(".name").filter(function(name){
        return name.indexOf(" ") >= 0;
      })
    }
  }
});

var me = new Person({name: "James"});

me.on("lastValidName", function(lastValid) {
  console.log(lastValid)
});

me.name = "JamesAtherton"; // No change

me.name = "James Atherton";
//-> console.logs "James Atherton";

me.name = "JustinMeyer"; // No change

me.name = "Justin Meyer";
//-> console.logs "Justin Meyer";
```
