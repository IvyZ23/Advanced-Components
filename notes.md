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

### Reactivity in Vue.js

Vue recursively converts its properties into getters/setters and makes variables reactive.
