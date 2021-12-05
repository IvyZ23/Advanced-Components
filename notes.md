## Advanced Components

---

### The Reactivity System

Vue knows that if one variable changes, any other variable that corresponds with it changes as well (for example, if the variable "price" changes from 10 to 20, then the variable "totalPrice" would also increase by 10)

However, JavaScript does not work like this, as it is procedural and not reactive, so it would require more to make it reactive.

```
let price = 5
    let quantity = 2
    let total = 0
    let target = null
    let storage = []

    function record () {
      storage.push(target)
    }

    function replay () {
      storage.forEach(run => run())
    }

    target = () => { total = price * quantity }

    record()
    target()

    price = 20
    console.log(total) // => 10
    replay()
    console.log(total) // => 40
```

The above code allows for the "total" variable to change when the variable "quantity" changes. This works by pushing the function "target" into the "storage" array by using the function "record". When the user wants to call the function "target" they would call the function "replay", which will loop over every function in the array called "storage".

**Dependency Classes**

Dependency classes can maintain a list of targets that get notifies when it gets called.

Instead of storage we’re now storing our anonymous functions in subscribers. **Instead of our record function we now call depend**and we now use notify instead of replay. To get this running:

```
class Dep { // Stands for dependency
      constructor () {
        this.subscribers = [] // The targets that are dependent, and should be
                              // run when notify() is called.
      }
      depend() {  // This replaces our record function
        if (target && !this.subscribers.includes(target)) {
          // Only if there is a target & it's not already subscribed
          this.subscribers.push(target)
        }
      }
      notify() {  // Replaces our replay function
        this.subscribers.forEach(sub => sub()) // Run our targets, or observers.
      }
    }

const dep = new Dep()

    let price = 5
    let quantity = 2
    let total = 0
    let target = () => { total = price * quantity }
    dep.depend() // Add this target to our subscribers
    target()  // Run it to get the total

    console.log(total) // => 10 .. The right number
    price = 20
    console.log(total) // => 10 .. No longer the right number
    dep.notify()       // Run the subscribers
    console.log(total) // => 40  .. Now the right number
```

This makes the code more reusable. Each varaible would need to have a Dep class.

**watcher function** = feature that allows one to watch a component and perform specified actions when the value of the component changes

Instead of writing it like this:

```
target = () => { total = price * quantity }
    dep.depend()
    target()
```

It can be written like this instead with a watcher function:

```
watcher(() => {
      total = price * quantity
    })
```

This watcher function would allow the price to change when quantity changes:

```
function watcher(myFunc) {
      target = myFunc // Set as the active target
      dep.depend()       // Add the active target as a dependency
      target()           // Call the target
      target = null      // Reset the target
    }
```

The watcher function takes a myFunc argument, sets that as a our global target property, calls dep.depend() to add our target as a subscriber, calls the target function, and resets the target.

**Object.defineProperty()**

This allows us to define **getter and setter** functions for a property.

```
let data = { price: 5, quantity: 2 }

    Object.defineProperty(data, 'price', {  // For just the price property

        get() {  // Create a get method
          console.log(`I was accessed`)
        },

        set(newVal) {  // Create a set method
          console.log(`I was changed`)
        }
    })
    data.price // This calls get()
    data.price = 20  // This calls set()
```

When the object is called, it would call the get function and when the object is changed it would call the set function.

The final code that is reactive:

```
let data = { price: 5, quantity: 2 }
let target, total, salePrice

    class Dep {
      constructor () {
        this.subscribers = []
      }
      depend() {
        if (target && !this.subscribers.includes(target)) {
          // Only if there is a target & it's not already subscribed
          this.subscribers.push(target)
        }
      }
      notify() {
        this.subscribers.forEach(sub => sub())
      }
    }

    // Go through each of our data properties
    Object.keys(data).forEach(key => {
      let internalValue = data[key]

      // Each property gets a dependency instance
      const dep = new Dep()

      Object.defineProperty(data, key, {
        get() {
          dep.depend() // <-- Remember the target we're running
          return internalValue
        },
        set(newVal) {
          internalValue = newVal
          dep.notify() // <-- Re-run stored functions
        }
      })
    })

    // My watcher no longer calls dep.depend,
    // since that gets called from inside our get method.
    function watcher(myFunc) {
      target = myFunc
      target()
      target = null
    }

    watcher(() => {
      data.total = data.price * data.quantity
    })

    watcher(() => {
      salePrice = data.price * 0.9
    })
```

**In Vue**
![-](/src/componenttree.png)

---

### Proxy

Instead of looping through each property to add getters/setters we can set up a proxy on our data object using:

```
//data is our source object being observed
    const observedData = new Proxy(data, {
      get() {
        //invoked when property from source data object is accessed
      },
      set() {
        //invoked when property from source data object is modified
      },
      deleteProperty() {
        //invoked when property from source data object is deleted
      }
    });
```

Implementing proxy can get rid of the stop of "set".

---

### Reactivity in Vue.js

Vue recursively converts its properties into getters/setters and makes variables reactive.

**Call Stack for Creating Reactivity in Vue**

1. Creating new vue instance in index.html
2. /src/core/instance/index.js
3. /src/core/instance/init.js
4. /src/core/instance/state.js
5. /src/core/instance/index.js

**Caveats**

1. Vue wraps observed array mutation methods to trigger updates (examples are when push, pop, shift, unshift, splice, sort, and reverse are used). However, Vue would not detect the following changes:

```
app.items[indexOfItem] = newValue
```

    - solution:

    ```
    Vue.set(app.items, indexOfItem, newValue)
    ```

```
app.items.length = newLength
```

    - solution:

    ```
    app.items.splice(indexOfItem, 1, newValue)
    ```

2. Vue cannot detect property addition or deletion

For example, this would not work:

```
var app = new Vue({
    el: "#app",
    data: {
        product: "Socks"
    }
})

app.color = "red"
```

A solution is to nest the object into another object:

```
data: {
    product: {
        name: "Socks"
    }
}

vue.set(app.product, "color", "red")
```

For multiple properties, a solution is:

```
app.product = Object.assign({}, app.product, {
    color: "red",
    price: 20
})
```

---

### Template Compilation and Render Functions

**Component Template Rendering**

This happens after reactivity is set up.

1. Compiles template into a render function. (can happen on the server or client side)
2. Run the render function, which will create a virtual DOM node (VNode) and loads what we see in the browser (only happens on client side)

**DOM (Document Object Model)**

It is the browser's interface (API) to change what is displayed on the screen

Frameworks like vue does the heavylifting when it comes to DOM Trees getting too big, but searching and updating the DOM can still be slow. Thus, we use the virtual DOM.

**Virtual DOM**

A way of representing the actual DOM with JS objects (like a blueprint for the actual DOM).

For example:

```
<div>Hello</div>
```

would be:

```
{
    tag:"div",
    children: [
        {
            text: "Hello"
        }
    ]
}
```

Creating an element using render:

```
<script>
   var app = new Vue({
       el: '#app',
       render(createElement) {
           return createElement('div', 'hello')
       }
   })
</script>
```

**JSX**

**Steps to Initialize Component and Insert VNode**

- Initialize events and lifecycle
  1. beforeCreate <----lifecycle hooks
- initialize injects and reactivity (state) 2. created <----lifecycle hooks
- if template exists, compile it into a render function 3. beforeMount <----lifecycle hooks
- start mounting (take VNode, aka the virtual DOM, and insert it into the DOM) 4. mounted <----lifecycle hooks

---

### Functional Components

**Major Uses**

1. cheap leaf componenets that can be reused by don't come witht he cost of a full component (one less instance)
2. using functional wrapper componenets (uses logic)

A component which holds no state and no instance. In more plain English, this means that the component does not support reactivity and cannot make a reference to itself through the this keyword.

It offers performance benefits, greater reusability, and allows for simpler code.

Functional Components can't have their own data, computed properties, watchers, lifecycle methods, or methods. They can't have templates unless it is precompiled froma single-file component. It can be passed props, attributes, and slots. It also returns a VNode or an array of VNodes from a render function.

To make a component functional:

```
<div id="app">
      <big-topic>
        Hiking Boots
      </big-topic>
    </div>

    <script src="vue.js"></script>
    <script>
      Vue.component('big-topic', {
      functional: true,  // <-----
      render(h, context) { // Notice the new context parameter
        return h('h1', context.slots().default)
      }
    })

      new Vue({
       el: '#app'
      })
    </script>
```

**_The "h" in the code above just stands for hyperscipt, or HTML_**

Above, the "context" parameter is used so that we can access slots, props, children, data, parent, and listeners.

In single file components, the component can be made functional by simply putting the "funtional" in the template tag:

```
**in component.vue file**
    <template functional>
        <h1>
            <slot></slot>
        </h1>
    </template>
```

During build time, the above would be compiled into a render function and can be used to speed up the application.

**Functional Wrapper Components**

Functional components are great to use when you need a way of programmatically delegating to a specific component.

Example from video:
![-](/src/functionalwrapper.jpg)
_In the picture, the component "SmartTable" decides whether or not NormalTable or EmptyTable is shown_

```
    <div id="app">
      <smart-table :items='vehicles'>
    </div>

    <script src="vue.js"></script>
    <script>
    const EmptyTable = {
      template: `<h1>Nothing Here</h1>`
    }
    const NormalTable = { // Normally this would be more complex
      template: `<h1>Normal Table</h1>`
    }

    Vue.component('smart-table', {
     functional: true,
     props: { items: { type: Array } },
     render(h, context) {
       if (context.props.items.length > 0 ) {  // Delegate
         return h(NormalTable, context.data, context.children)
       } else {
         return h(EmptyTable, context.data, context.children)
       }
     }
    })
    new Vue({
     el: '#app',
     data: {
      vehicles: [ 'Fiat', 'Toyota', 'BMW' ]
     }
    })
    </script>
```

With the `h(NormalTable, context.data, context.children)` line we’re rendering the component and passing through all our data which includes attributes, event listeners, and props so Normal table will have access to them.

**Destructuring with ES2015 Destructuring**

Simple example of destructuring:

```
//WITHOUT DESTRUCTURING
var fullName = {
    first: "Gregg",
    last: "Pollack"
}

function printName(nm) {
    console.log(nm.first + " " + nm.last)
}

//WITH DESTRUCTURING
function printName({ first, last }) {
    console.log(first + " " + last)
}
```

So, with the example from before about the SmartTable:

```
//WITHOUT DESTRUCTURING
    render(h, context) {
        if (context.props.items.length > 0 ) {  // Delegate
            return h(NormalTable, context.data, context.children)
        } else {
            return h(EmptyTable, context.data, context.children)
        }
    }

//WITH DESTRUCTURING
    render(h, { props, data, children }) {
       if (props.items.length > 0 ) {
         return h(NormalTable, data, children)
       } else {
         return h(EmptyTable, data, children)
       }
     }
```

---

### Internal Mounting

Vue takes our functions and turns them into render functions with shortcuts:

```
target._s = toString
target._v = createTextVNode
...
```

_^^^ found in /src/core/instance/render-helpers.js/_

These shortcuts make our render short cuts as small as possible.

All DOM functions are called in node-ops.

**Directory for each part of our application:**
/src/core/instance/ = Vue components
/src/platforms/ = Web browser
/src/core/vdom = Virtual DOM (VNode)

---

### Scoped Slots and Render Props

Using scoped slots when you need components to have some differences (such as, for the example below, one of the component needs an image):

```
<div id="app">
      <products-list :products="products"></products-list>
      <products-list :products="products">
        <template slot="product" slot-scope="slotProps">
          <img :src="slotProps.product.image" /> {{ slotProps.product.name.toUpperCase() }}
        </template>
      </products-list>
    </div>
    <script src="vue.js"></script>
    <script>
      Vue.component('products-list', {
        props: {
          products: {
            type: Array,
            required: true
          }
        },
        template: `
        <ul>
          <li v-for="product in products">
            <slot name="product" :product="product" >
                {{ product.name }}
            </slot>
          </li>
        </ul>`
      })
      new Vue({
          el: '#app',
        data: {
          products: [{
            name: 'Magnifying Glass',
            image: 'magnify.png'
          }, {
            name: 'Light Bulb',
            image: 'bulb.png'
          }]
        }
       })
    </script>
```

Another way to do it is with render props (instead of using a template for the component, we use a render function):

```
<div id="app">
      <products-list :products="products"></products-list>
    </div>
    <script src="vue.js"></script>
    <script>
      Vue.component('products-list', {
        props: {
          products: {
            type: Array,
            required: true
          }
        },
        render(h) {  // <-- Notice our render function
          return h('ul', [
            this.products.map(product =>
              h('li', [product.name])
            )
          ])
        }
      })
      new Vue({ ... Same as above ... })
    </script>

```

To use Render props as a technique to solve our problem, we will be literally sending in a render function as a prop into our component. See below:

```
<div id="app">
      <products-list :products="products"></products-list>
      <products-list :products="products" :product-renderer="imageRenderer"></products-list>
    </div>
    <script src="vue.js"></script>
    <script>
      Vue.component('products-list', {
        props: {
          products: {
            type: Array,
            required: true
          },
          productRenderer: {  // <-- Here's our new prop
            type: Function,
            default (h, product) { // <-- By default just print the name
              return product.name
            }
          }
        },
        render(h) {
          return h('ul', [
            this.products.map(product =>
              h('li', [this.productRenderer(h, product)]) // use our new prop
            )
          ])
        }
      })
      new Vue({
        el: '#app',
        data: {
          products: [{
            name: 'Magnifying Glass',
            image: 'magnify.png'
          }, {
            name: 'Light Bulb',
            image: 'bulb.png'
          }],
          imageRenderer(h, product) { // <-- The imageRenderer I'm passing in
            return [
              h('img', {
                attrs: {
                  src: product.image
                }
              }),
              ' ',
              product.name.toUpperCase()
            ]
          }
        }
      })
    </script>
```

The above code works like so:
The image information in the new Vue instance in "imageRenderer" would be sent to the HTML, where it sends in data from "imageRenderer" to "product-renderer". The product-renderer will then take that image information and send it to the render function to be processed, therefore sending the picture to the DOM.
