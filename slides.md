---
theme: default
highlighter: shiki
transition: slide-left
lineNumbers: true
layout: cover
---

# Building reactive systems with Proxies and Reflects

Re-calculating observability with meta programming

Yash Dave


---
layout: intro
---

# Yash Dave

<ul>
<li>Product Engineering, DeepSource</li>
<li>Keyboard enthusiast</li>
<li>Rocket league, Apex Legends, etc.</li>
</ul>

<div class="my-10 grid grid-cols-[28px,1fr] w-min gap-y-4">
  <lucide-github class="opacity-80"/>
  <div><a href="https://github.com/amorpheuz" target="_blank">amorpheuz</a></div>
  <lucide-twitter class="opacity-80"/>
  <div><a href="https://twitter.com/amorpheuz" target="_blank">amorpheuz</a></div>
  <lucide-user class="opacity-80"/>
  <div><a href="https://amorpheuz.dev" target="_blank">amorpheuz.dev</a></div>
</div>

<img src="https://amorpheuz.dev/images/yash_512.jpg" class="rounded-full w-64 abs-tr mt-24 mr-14"/>

---
layout: default
---

# What's reactivity?

Let's see this simple piece of code

```js {1-4,6-8|5,9|all}
let i = 10;
let j = 20;
let x = i + j;
console.log(x); 
// 30

j = 30;
console.log(x) 
// still 30
```

<div v-click>

If it was reactive, the outputs would have been:

```js {5,9}
let i = 10;
let j = 20;
let x = i + j;
console.log(x); 
// 30

j = 30;
console.log(x) 
// it's 40 now
```

</div>

<!-- 
- Focus on outputs
-->
---
layout: default
---

# What to do then?

- No way to track reading and writing of local variables.
- So, we intercept the reading and writing of object properties.
- Vue does it in two ways:
  - `getter` / `setters`
  - `Proxies`

## What's a `Proxy`?

- Enables creation of a proxy for another object.
- Acts as a replacement for the original object, but allows redefining basic operations.
- This includes getting, setting and defining properties.
- It's initialized with **two** parameters:
  - `target`: the original object to be proxied
  - `handler`: object that defines the operations intercepted

<!-- 
- Troubles with getter/setter, cumbersome to use; less performant.
- Troubles with proxies, destructuring from a proxy losses reactivity due to loss of reference.
- Also, `===` wont work since its a proxy that's returned.
- List of internals:
  - `getPrototypeOf()`
  - `setPrototypeOf()`
  -	`isExtensible()`
  - `preventExtensions()`
  -	`getOwnPropertyDescriptor()`
  -	`defineProperty()`
  -	`has()`
  -	`get()`
  -	`set()`
  -	`deleteProperty()`
  -	`ownKeys()`
-->

---
layout: default
---

# Observing changes

```js {all|3-12|4-7|8-11|14-17|all}
const someData = {name: 'Yash', locCommitted: 256}

const dataHandlers = {
  get(data, propName) {
    console.log(`Getting ${propName}: ${data[propName]}`)
    return data[propName]
  },
  set(data, propName, value) {
    console.log(`Setting ${propName}: ${value}`)
    data[propName] = value
  }
  ...
}

const dataProxy = new Proxy(someData, dataHandlers)

console.log(dataProxy.name)
// Yash
```

But, is there a better way to perform all methods available on `Object`?

<!-- 
- Focus on all -> handlers -> get -> set -> proxy creation
-->

---
layout: default
---

# `Reflect`s are the key

- Built-in object that provides methods for interceptable JavaScript operations.
- Same methods as those needed by `Proxy` handlers.
- Not a function object, not constructible.
- Provides static methods like:
  - `Reflect.get()`
  - `Reflect.set()`
  - `Reflect.has()`
  - `Reflect.ownKeys()`
  - ...

Let's revisit our handler then.

<!-- 
- The primary use case of the Reflect object is it to make it easy to interfere functionality of an existing object with a proxy and still provide the default behavior.
- These methods are very convenient because you don't have to think of syntactic differences in JavaScrict for specific operations and can just use the same method defined in Reflect when dealing with proxies.
- List of internals:
  - `getPrototypeOf()`
  - `setPrototypeOf()`
  -	`isExtensible()`
  - `preventExtensions()`
  -	`getOwnPropertyDescriptor()`
  -	`defineProperty()`
  -	`has()`
  -	`get()`
  -	`set()`
  -	`deleteProperty()`
  -	`ownKeys()`
-->

---
layout:default
---

# Observing changes with `Reflect`

```js {all|6,10}
const someData = {name: 'Yash', locCommitted: 256}

const dataHandlers = {
  get(data, propName) {
    console.log(`Getting ${propName}: ${data[propName]}`)
    return Reflect.get(data, propName)
  },
  set(data, propName, value) {
    console.log(`Setting ${propName}: ${value}`)
    Reflect.set(data, propName, value)
  }
  ...
}

const dataProxy = new Proxy(someData, dataHandlers)

dataProxy.locCommitted = 258
console.log(dataProxy.locCommitted)
// 258
```

---
layout: default
---

# Let's define a problem

```js {1-2|4-8|9-14|15-17|all}
// I have this data
const someData = {name: 'Yash', lineAdditions: 10, lineDeletions: 5}
 
// My side effects
const depAdditions = () => console.log(`additions = ${someData.lineAdditions}`)
const depDeletions = () => console.log(`deletions = ${someData.lineDeletions}`)
const depNet = () => console.log(`netChange = ${someData.lineAdditions - someData.lineDeletions}`)
  
// Running the side effects
depAdditions()
depDeletions()
depNetz()
// output: additions = 10, deletions = 5, netChange = 5
  
// mutate data
proxy.lineAdditions = 20
// ??? - No updates
```

---
layout: default
---

# Now, how do we track and trigger the changes?

- A global `WeakMap<target, Map<key, Set<effect>>>` by Vue data structure is used to track effect subscriptions.


```js {0|1-14|17-19|21-30|32-41|42-51} {maxHeight:'225px'}
const dependentMap = new WeakMap()
  
// track a property
const trackUpdates = (_type, data, propName) => {
  if (isDryRun && currentFn) {
    if (!dependentMap.has(data)) {
      dependentMap.set(data, new Map())
    }
    if (!dependentMap.get(data).has(propName)) {
      dependentMap.get(data).set(propName, new Set())
    }
    dependentMap.get(data).get(propName).add(currentFn)
  }
}

// trigger a change in property
const trigger = (_type, data, propName) => {
  dependentMap.get(data).get(propName).forEach(triggerFunction => triggerFunction())
}

// Setting up an observer
let isDryRun = false
let currentFn = null
const observe = (fn) => {
  isDryRun = true
  currentFn = fn
  fn()
  currentFn = null
  isDryRun = false
}

// Updated handlers
const handlers = {
  get(...args) { 
    trackUpdates('get', ...args); 
    return Reflect.get(...args);
  },
  has(...args) { 
    trackUpdates('has', ...args);
    return Reflect.has(...args) 
  },
  set(...args) { 
    Reflect.set(...args); 
    trigger('set', ...args) 
  },
  deleteProperty(...args) {
    Reflect.deleteProperty(...args);
    trigger('delete', ...args)
  },
  // ...
}
```

---
layout: default
---

# Take it for a spin

```js
const someData = {name: 'Yash', lineAdditions: 10, lineDeletions: 5}
const proxy = new Proxy(someData, handlers)
 
// observe functions
const depAdditions = () => console.log(`additions = ${proxy.lineAdditions}`)
const depDeletions = () => console.log(`deletions = ${proxy.lineDeletions}`)
const depNet = () => console.log(`netChange = ${proxy.lineAdditions - proxy.lineDeletions}`)
  
// dry-run all dependents
observe(depAdditions)
observe(depDeletions)
observe(depNet)
// output: additions = 10, deletions = 5, netChange = 5
  
// mutate data
proxy.lineAdditions = 20
// output: additions = 20, netChange = 15
```

---
layout: statement
---

# 🎉

---
layout: statement
---

# Questions?


<div class="mt-10 space-y-2">
<strong>Later? Reach out at</strong>
<div class="grid grid-cols-[28px,1fr] w-min gap-y-4 mx-auto text-left">
  <lucide-twitter class="opacity-80"/>
  <div><a href="https://twitter.com/amorpheuz" target="_blank">amorpheuz</a></div>
</div>
</div>