# Embedding a Javascript or Typescript component into your Rust project

If you need to integrate a scripted component into your latest project, don't worry; it does not need to be a complicated procedure.

## The Javascript v8 runtime
The v8 runtime has been brought to the Rust ecosystem by the [Deno project](https://deno.com/).

However, the v8 engine it relies on (and by extension the Deno core itself) comes equipped with its fair share of pitfalls and gotchas.

Enter rustyscript - a Deno API wrapper designed to abstract away the v8 engine details, and allow you to operate directly on Rust types when working with javascript modules.

## Rustyscript -- Effortless JS integration for Rust
[The rustyscript crate](https://crates.io/crates/rustyscript) is how I will be using the javascript runtime in this article - it is a Deno API wrapper for Rust that aims to prevent the common pitfalls and complications of using the v8 engine from Rust.

It will take care of details such as: 
- Transpiling typescript
- Resolving asynchronous JS
- Allowing modules to import one another
- Deserializing return values back into Rust types.
- Setting up Deno's extensions

rustyscript will also sandbox the code from the host machine by default, blocking access to the filesystem, as well as network and timer resources.

## Our first runtime

Let's spin up a basic JS runtime, and try to run some javascript.

First, we will need some JS - let's use typescript in this example. *It's important to know that when transpiling, rustyscript will not perform type-checking. If you want to preserve strong-typing you will need to check the argument types yourself.*

The typescript below - let's call it `get_value.ts` - sets up a simple API; one function sets up an internal value, and another retrieves that value:

```typescript
let my_internal_value: number;

export function getValue(): number {
  return my_internal_value;
}

export function setValue(value: number) {
  my_internal_value = value * 2;
}
```

Now from the Rust side, we spin up a runtime, and import our file:

```rust
let mut module = rustyscript::import("get_value.ts")?;
```

No really, that's it - a working JS runtime with our shiny new module imported and ready to go - We can now call our module's API at will.

First, we set up our module's internal value by calling `setValue(5)` - the `::<Undefined>` here just means we don't care about what the function returns:

```rust
use rustyscript::{json_args, Undefined};

module.call::<Undefined>("setValue", json_args!(5))?;
```

`json_args!` is a macro taking in a comma-separated list of Rust primitives to turn them into serialized values we can send to javascript.

Now we can get our value back out. 

Since now we do care about what type we get, we tell the compiler to deserialize the JS function's return value as an i64. We will get an `Error::JsonDecode` if the wrong type is returned by javascript:

```rust
let value: i64 = module.call("getValue", json_args!())?;
```

Putting it all together we get this:

```rust
use rustyscript::{json_args, import, Undefined, Error};

fn main() -> Result<(), Error> {
  let mut module = rustyscript::import("get_value.ts")?;

  module.call::<Undefined>("setValue", json_args!(5))?;
  let value: i64 = module.call("getValue", json_args!())?;
  assert_eq!(value, 10);
}
```

## A more comprehensive example

But one ES module does not an ecosystem make. So let's try it again, but this time we will add more options to our runtime, and manage our errors a little better.

Let's embed our TS module into the executable directly; after all, it is a very small file we will always need - why take the extra hit from filesystem overhead?

```rust
use rustyscript::{module, StaticModule};

const API_MODULE: StaticModule = module!(
  "get_value.ts",
  "
    let my_internal_value: number;
    
    export function getValue(): number {
      return my_internal_value;
    }
    
    export function setValue(value: number) {
      my_internal_value = value * 2;
    }

    export default setValue;
  ");
```

Next, we need a runtime. There are lots of options available here but the defaults will work for us here:

```rust
use rustyscript::{json_args, Error, Runtime, RuntimeOptions, Undefined};

fn main() -> Result<(), Error> {
  let mut runtime = Runtime::new(RuntimeOptions::default())?;
```

Now we can include our static module:

```rust
  let module_handle = runtime.load_module(&API_MODULE.to_module())?;
  runtime.call_entrypoint::<Undefined>(&module_handle, json_args!(2))?;

  Ok(())
}
```

The `call_entrypoint` function will call our module's setValue function for us - the module's default export was found and a reference to it stored in advance on load so that this function call can be made with less overhead.

Just like before, `::<Undefined` means we do not care if the function returns a result.

Now that we have a runtime, let's add a second module that can make use of it! We'll name this file `use_value.js`

```javascript
import * as get_value from './get_value.ts';

// We will get the value set up for us by the runtime, and transform it
// into a string!
let value = get_value.getValue();
export const final_value = `$${value.toFixed(2)}`;
```

Now let's add the following to `main()`, right before the `Ok(())` at the bottom:

```rust
let use_value_handle = runtime.load_module(&Module::load("examples/medium.js")?)?;
let final_value: String = runtime.get_value(Some(&use_value_handle), "final_value")?;
```

We load our new module from the filesystem - the handle that `load_module` returns is used to give context to future calls.

We use that returned handle to extract the const that it exports, and then we tell the compiler we'd like it as a string.

Finally, we can check that we received back the value we expected:

```rust
println!("The received value was {final_value}");
```

When we run our completed example, we should see a print out to the console:

> The received value was $4.00

Our static module was able to be imported for use by another JS module, and the JS side's API was able to be accessed by our Rust backend.

The final code looks like this:

```rust
use rustyscript::{json_args, Error, Runtime, RuntimeOptions, Undefined};
use rustyscript::{module, StaticModule};

const API_MODULE: StaticModule = module!(
  "get_value.ts",
  "
    let my_internal_value: number;
    
    export function getValue(): number {
      return my_internal_value;
    }
    
    export function setValue(value: number) {
      my_internal_value = value * 2;
    }

    export default setValue;
  ");

fn main() -> Result<(), Error> {
  let mut runtime = Runtime::new(RuntimeOptions::default())?;

  let module_handle = runtime.load_module(&API_MODULE.to_module())?;
  runtime.call_entrypoint::<Undefined>(&module_handle, json_args!(2))?;

  let use_value_handle = runtime.load_module(&Module::load("examples/medium.js")?)?;
  let final_value: String = runtime.get_value(Some(&use_value_handle), "final_value")?;

  println!("The received value was {final_value}");
  Ok(())
}
```

## Conclusion

Embedding a JS or TS component into Rust does not have to be difficult or time-consuming.

By leveraging API wrappers like rustyscript we can include a full-fledged runtime with very little learning curve or time investment. 

Over a dozen more examples can be found on [rustyscript's GitHub repository](https://github.com/rscarson/rustyscript)
