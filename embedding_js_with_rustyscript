# Embedding a Javascript or Typescript component into your Rust project using rustyscript
If you've got a need to integrate a scripted component into your latest project - do not fret; it does not need to be a complicated procedure.

## The Javascript v8 runtime
The v8 runtime has been brought to the Rust ecosystem by the [Deno project](https://deno.com/), but the v8 engine, and by extension the deno_core crate that integrates it, comes equipped with its fair share of pitfalls and gotchas.

Enter rustyscript - a Deno API wrapper designed to abstract away the v8 engine details, and allow you to operate directly on Rust types when working with javascript modules.

## Enter rustyscript
[The rustyscript crate](https://crates.io/crates/rustyscript) is how I will be using the javascript runtime in this article - it is a Deno API wrapper for Rust that aims to prevent the common pitfalls and complications of using the v8 engine from Rust.

It will automatically transpile typescript, resolve asynchronous JS, allow modules to import one another, and handle deserialization of returns back into Rust types.
Additionally, if security is a concern, rustyscript will by default sandbox the code from the host machine, by remove access to filesystem, network, and timers.
Our first runtime

Let's spin up a basic JS runtime, and try to run some javascript.

First, we will need some JS - let's use typescript in this example. *It's important to know that when transpiling, rustyscript will not perform type-checking. If you want to preserve strong-typing you will need to check the argument types yourself.*

The tyepscript below - let's call it `get_value.ts` - sets up a simple API; one function sets up an internal value, another retrieves that value:

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
let mut module = rustyscript::import("get_value.ts")
  .expect("could not import the module!");
```

No really, that's it - a working JS runtime with our shiny new module imported and ready to go. We could improve this with better error management - most rustyscript functions return a `rustyscript::Error` that will hold more details of what went wrong.

We can now call our module's API at will.

First we set up our module's internal value by calling `setValue(5)` - the `::<Undefined>` here just means we don't care about what the function returns:

```rust
use rustyscript::{json_args, Undefined};

module.call::<Undefined>("setValue", json_args!(5))
  .expect("could not call JS function");
```

`json_args!` is a macro taking in a comma separated list of Rust primitives to
turn them into serialized values we can send to javascript.

Now we can get our value back out. Here we do care about what type we get, so we tell the compiler to deserialize the JS function's return value as an i64. We will get an `Error::JsonDecode` if the wrong type is returned by javascript:

```rust
let value:i64 = module.call("getValue", json_args!())
  .expect("could not call JS function");
```

## A more comprehensive example
But one ES module does not an ecosystem make. So let's try it again, but this time we will add more options to our runtime, and manage our errors a little better.

This time we will embed this module into the executable directly. After all, it is a very small file we will always need - why take the extra overhead for the filesystem!

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
  ");
```

Next we need a runtime. There are a handful of options available here but the one we need right now is default_entrypoint.

This tells the runtime that a function is needed for initial setup of our runtime:

```rust
use rustyscript::{json_args, Error, Runtime, RuntimeOptions, Undefined};

fn main() -> Result<(), Error> {
  let mut runtime = Runtime::new(RuntimeOptions {
    default_entrypoint: Some("setValue".to_string()),
    ..Default::default()
  })?;
```

Now we can include our static module - `to_module()` is needed to pull into into a form we can use.

The `call_entrypoint` function will call our module's setValue function for us - the function was found and a reference to it stored in advance on load so that this function call can be made with less overhead.

Just like before, `::<Undefined` means we do not care if the function returns a result.

```rust
  let module_handle = runtime.load_module(&API_MODULE.to_module())?;
  runtime.call_entrypoint::<Undefined>(&module_handle, json_args!(2))?;

  Ok(())
}
```

Now that we have a runtime, let us add a second module that can make use of it! We shall name this file `use_value.js`

```javascript
import * as get_value from './get_value.ts';

// We will get the value set up for us by the runtime, and transform it
// into a string!
let value = get_value.getValue();
export const final_value = `$${value.toFixed(2)}`;
```

Now let's add the following to `main()`, right before the `Ok(())` at the bottom.

We load our new module from the filesystem. The handle that `load_module` returns is used to give context to future calls.

We use the returned handle to extract the const that it exports, and then we tell the compiler we'd like it as a string:

```rust
let use_value_handle = runtime.load_module(&Module::load("examples/medium.js")?)?;
let final_value: String = runtime.get_value(&use_value_handle, "final_value")?;
```

Finally, we can check that we received back the value we expected:

```rust
println!("The received value was {final_value}");
```

When we run our completed example, we should see a print out to the console:

> The received value was $4.00

Our static module was able to be imported for use by another JS module, and the JS side's API was able to be accessed by our Rust backend.

## Conclusion
Embedding a JS or TS component into Rust does not have to be difficult or time consuming. By leveraging API wrappers like rustyscript we can include a full-fledged runtime with very little learning curve or time investment. 

This example and more can be found on [rustyscript's github repository](https://github.com/rscarson/rustyscript)
