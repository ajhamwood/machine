# MACHINE.js

App state, events, and async chronology management tools for client-side JS in about 2kb.

### Example usage

HTML:

```html
<main></main>
<template id='my-title'><h1></h1></template>
<script src='https://cdn.jsdelivr.net/gh/ajhamwood/machine@master/machine.min.js'></script>
<script src='my-app.js'></script>
```

JS:

```js
// Page state
var app = new $.Machine({
  title: '~Hello world~'
});

// Events
$.targets({
  load () {
    app.emit('init')
  },
  app: {
    init () {
      let main = $.load('my-title', 'main')[0][0];
      main.textContent = this.title;
      $.queries({
        h1: {
          'touchstart mouseover' () {
            main.classList.add('active')
          },
          'touchend mouseout' () {
            main.classList.remove('active')
          }
        }
      })
    }
  }
})
```

## Documentation

### `$(query, root)`, `$.all(query, root)`

Improved `querySelector`/`querySelectorAll`. Query the DOM tree and return the first matching `Element` or null, or with `$.all` an `Array` of the resulting `Element`s. Optionally include a root node to query within (default is `document`).

```js
// Returns the 5th element with class 'cell' inside a 'main' element.
let cell = $('main .cell')

// Inserts a text node with the index of the label into all labels inside the cell element.
$.all('label', cell).forEach((el, i) => el.textContent = i)
```

### `$.load(template, query, root)`

Improved `importNode`. Insert the contents of the template selected by id into each query result, and return an `Array` of `Array`s of the resulting `Element`s. Optionally include a root node to query within (default is `document`).

HTML:

```html
<main></main>
<template id='cell'>
  <div class='cell'></div>
<template>
<template id='label'>
  <label></label>
  <input type='radio' name='main'>
</template>
```

JS:

```js
// Places 5 copies of the contents of the cell template into main
for (let i = 0; i < 5; i++) $.load('cell', 'main')

// Adds input elements and their labels to each cell in main.
$.load('label', '.cell').forEach(([lbl, inp], i) => {
  lbl.setAttribute('for', 'cell' + i);
  lbl.textContent = i;
  inp.setAttribute('id', 'cell' + i)
})
```

### `$.loadWc(tag, { constructor, callback1, callback2, customMethod, ...  }, attributes)`

Succinct web components. Provide the contents to the web component in a template with your tag as id, then call with the tag name, a hash including the element constructor, any lifecycle callbacks or other instance methods, and optionally a list of element attribute names.

HTML:

```html
// Adapted from https://github.com/vanillawc/wc-marquee
<wc-marquee>Hello</wc-marquee>
<template id="wc-marquee">
  <style>
    .marquee { margin: 0 auto }
    .marquee span {
      display: inline-block;
      padding-left: 100%;
      animation: marquee 15s linear infinite }
    @keyframes marquee {
      0% { transform: translate(0, 0) }
      100% { transform: translate(-100%, 0) } }
  </style>
  <p class="marquee" style="width: inherit;"><span><slot></slot></span></p>
</template>
```

JS:

```js
// Creates a marquee element
$.loadWc("wc-marquee", {
  constructor () { const el = $(".marquee", this.shadowRoot)[0] },
  connectedCallback () { this.style.width ??= '100%' }
})
```

### `$.queries({ query: { listener1: function fn () {...}, listener2: 'fn', ... }, ... }, root)`

Improved `addEventListener`/`removeEventListener`, for use with `Element`s. Provide a function to add to a given element's event listeners, or provide the name of a function to remove from its event listeners. Multiple listeners can be mass assigned/deleted by separating their names with a space in the key. The targeted element is available in the listener as the local `this`. Optionally include a root node to query within (default is `document`).

```js
// Inverts the checked state of each radio input when its label is sent a click or a touchstart event
$.queries({
  label: {
    'click touchstart' (e) {
      e.preventDefault();
      this.control.checked = !this.control.checked
    }
  }
})

// Removes the touchstart event listener on all labels (its function is named 'click touchstart' in the
// code above)
$.queries({ label: { touchstart: 'click touchstart'} })
```

### `new $.Machine({ property: value, ... })`

Improved `EventEmitter`. Construct a Machine object which has `on`, `stop`, `emit`, `emitAsync`, and `state` methods, with the argument object as state. The state object is sealed, so properties can be mutated but not added or deleted. The state is also available on the Machine instance.

```js
// Creates a new Machine with a single key 'selected'.
var app = new $.Machine({ selected: null })
// app.selected === null
```

### `$.Machine.on(listenerName, function)`

Add an event listener to the Machine. Both state and Machine methods are available in the listener as the local `this`.

```js
// Adds event listeners to app: one which set the key's value to its argument, one which clears the key's
// value to null, and one which waits some milliseconds.
app
  .on('select', function (i) { return this.selected = i })
  .on('clear', function c () { return this.selected = null })
  .on('wait', ms => new Promise(r => setTimeout(() => r(app.state()), ms)))
```

### `$.Machine.stop(listenerName, functionName)`

Remove an event listener from the Machine. Optionally provide the name of the function to remove one of multiple listeners on an event (defaults to the name of the event).

```js
app.stop('clear', 'c')
```

### `$.Machine.emit(listenerName, arg1, arg2, ...)`

Emit an event. If multiple listeners are attached, they are called sequentially in order of attachment. A hash of each listener's returned value indexed by name of function is returned.

```js
let sel = app.emit('select', 3).select
```

### `$.Machine.emitAsync(listenerName, arg1, arg2, ...)`

Emit an asynchronous event. If multiple listeners are attached, they are called sequentially in order of attachment. The resolved result is the same as in the synchronous version.

```js
// Waits 1s and prints the 'selected' value to console.
app.emitAsync('wait', 1000).then(({ wait }) => console.log(wait.selected))
```

### `$.Machine.state()`

Return the state object for the Machine.

```js
let sel = app.state().selected
```

### `$.targets({ listener1 () {...}, property1: { listener1a () {...}, property1a: {...}, ... }, ... }, object)`

Improved `addEventListener`/`removeEventListener`, for use with `EventTarget` and `$.Machine` objects. Provide a function to add to a given object's event listeners, provide the name of a function to remove from its event listeners, or provide an object to hierarchically add/remove event listeners on the given property of the current object. Multiple listeners can be mass assigned/deleted by separating their names with a space in the key. Multiple sub-objects can be targeted by giving a regex in the key. The targeted object is available in the listener as the local `this`. Optionally include a base object to locate properties/listeners within (default is `window`).

```js
// Creates two Machines named app1 and app2, adds an event listener to the window resize event, and a
// resize event to both Machines.
//
// NB: You must instantiate the Machines in the root context using `var` to have them appear as properties
// on the window object.

var app1 = new $.Machine({ width: null, height: null }),
    app2 = new $.Machine({ width: null, height: null });
$.targets({
  resize () { app.emit('resize', window.innerWidth, window.innerHeight) },
  'app\d': {
    resize (width, height) { Object.assign(this, { width, height }) }
  }
})
```

### `$.pipe(name, function1, [function2, function3, ...], ...)`

Improved `Promise.all`/`Promise.race`. Provide functions to run concurrently before passing on the last one completed; provide `Array`s of functions to run concurrently before passing on the first one completed.

```js
// Waits 1s before sending the 'nextSelected' event to app, once.
$.pipe('app-pipe', [() => app.emitAsync('wait', 2000), () => app.emitAsync('wait', 1000)]);
$.pipe('app-pipe', () => app.emit('nextSelected'))
```
