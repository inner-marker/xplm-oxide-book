# Flight Loops

We may want to accomplish some tasks occasionally while the X-Plane simulator and our plugin are running.

X-Plane provides a mechanism called the _flight loop_.

## Creating a Flight Loop

We will continue filling out our barebones example.

### Define the Flight Loop Field

First, we need to define a field in our plugin's struct to hold the flight loop.

```rust
#[xplm_plugin]
pub struct DemoPlugin {
  flight_loop: Option<FlightLoop>,
}
```

### Initialize the Flight Loop Field

Next, we need to impliment our own plugin and add a `new()` function. In the `new()` function, we will initialize the `flight_loop` field to `None`. We will set it up in the `plugin_start()` function.

```rust
impl DemoPlugin {
  fn new() -> Self {
    Self {
      // initialize the flight_loop field to None. We will set it up in plugin_start.
      flight_loop: None,
    }
  }
}
```

### Register the Flight Loop Callback

A callback function is a function that X-Plane will call at a specified interval. We will define a callback function in our plugin. Here is what that loops like. Next we will break down what all is happening here.

```rust
fn plugin_start(&mut self) {
  // Create a flight loop and register its callback
  let flight_loop_cb: FlightLoop = FlightLoop::create_plugin_fn(
    FlightLoopPhase::AfterFlightModel,
    Self::flight_loop_callback,
    plugin_state,
  ).expect("Failed to create flight loop");
  flight_loop_cb.schedule_seconds(1.0, true);
  self.flight_loop = Some(flight_loop_cb);
}
```

First, note that we are in the `plugin_start()` function.

We define a local variable to temporarily hold the flight loop callback. `let flight_loop_cb = FlightLoop::create_plugin_fn(...)`.

There are three ways to create a flight loop callback:

- `create(...)` - uses a closure to process information every flight loop.
- `create_fn(...)` - Specifies a function that does **not** have access to our plugin struct's fields. If you do not need to store any state from frame to frame, this is the way.
- `create_plugin_fn(...)` - Specifies a callback for a function that has access to our plugin's state at each callback. In other words, the callback has access to any of the `DemoPlugin` struct's fields.

In this example, we used the `create_plugin_fn(...)` method. It takes three arguments:

- An `Enum` for the `FlightLoopPhase`. This is an `Enum` that specifies when the callback should be called. In this case, we want it to be called after the flight model has been updated. The X-Plane API documentation cautions that some functions should only be called before or after the flight model is run. For instance, [`XPLMBeginWeatherUpdate()`](https://developer.x-plane.com/sdk/XPLMWeather/#XPLMBeginWeatherUpdate) from the Weather API should only be called before the flight model is run.
- A function pointer to the callback function. In this case, we are using `Self::flight_loop_callback`. We will see where we put this function in a moment.
- A pointer to the plugin's state. This macro is created by the `#[xplm_plugin]` attribute macro, so we do not need to create it ourselves.

The `create_plugin_fn(...)` function returns a `Result<FlightLoop, FlightLoopError>`. We use the `expect()` method to panic if the flight loop could not be created. You could use some other error handling method if you prefer. For this example, we are OK with a panic at launch.

Next, we need to schedule the first run of the flight loop. We have a few options for scheduleing the first run of the flight loop:

- `schedule(interval: f32, relative_to_now: bool)` - Schedules the flight loop to run after either a specified number of seconds or loops. Whether the callback is called in seconds or loops depends on whether the number is positive, negative, or zero. If the number is positrive, the callback will run in some number of seconds. If the numer is negative, the callback will run in some number of flight loops. If the number is zero, the callback will not run until it is scheduled again.
- `schedule_seconds(interval: f32, relative_to_now: bool)` - Schedules the flight loop to run after a specified number of seconds.
- `schedule_loops(interval: u32, relative_to_now: bool)` - Schedules the flight loop to run after a specified number of loops.

In this example, we scheduled the flight loop callback to run after 1 second.

Finally, we store the flight loop callback in our plugin's `flight_loop` field.

### Flight Loop Callback Function

The final step in impliemnting a flight loop is to define the callback function. This function will be called for the firs time at the interval we specified in the `schedule_seconds()` method. THen, thereafter, it will be called at the interval we return from the callback function.

The callback function takes three (plus one) arguments:

- `&mut self` - A reference to our plugin's struct. This is how we can access any of the fields in our plugin's struct.
- `_elapsed_since_last_call: f64` - The number of seconds that have elapsed since the last time this callback was called.
- `_elapsed_since_last_flight_loop: f64` - The number of seconds that have elapsed since the last flight loop.
- `_counter: i32` - A counter that increments each time the callback is called.

The callback has one return, an `f32`. This return value is the number of seconds until the callback is called again. If you return a negative number, the callback will be called after that many flight loops. If you return zero, the callback will not be called again until it is scheduled again.

```rust
fn flight_loop_callback(&mut self, _elapsed_since_last_call: f64, _elapsed_since_last_flight_loop: f64, _counter: i32) -> f32 {
    // Do some stuff here.

    1.0 // return an f32.
}
```

## Full Example

```rust
// src/lib.rs
use xplm_oxide::*;

// Define the struct for our plugin, using the `#[xplm_plugin]` attribute macro.
#[xplm_plugin]
pub struct DemoPlugin {
  flight_loop: Option<FlightLoop>,
}

// Impliment our Plugin
impl DemoPlugin {
  // Initialize all ifelds. FLight lop fields are initialized to `None`.
  fn new() -> Self {
    Self {
      // initialize the flight_loop field to None. We will set it up in plugin_start.
      flight_loop: None,
    }
  }

  // Fliight loop callback with the required signature.
  fn flight_loop_callback(&mut self, _elapsed_since_last_call: f64, _elapsed_since_last_flight_loop: f64, _counter: i32) -> f32 {
    // Do some stuff here.

    1.0 // return an f32.
  }
}

// Impliment the Plugin trait for our plugin struct.
impl Plugin for DemoPlugin {
    /// Required function called when the plugin is first loaded by X-Plane.
    fn plugin_start(&mut self) {
      // Create a flight loop and register its callback
      let flight_loop_cb = FlightLoop::create_plugin_fn(
        FlightLoopPhase::AfterFlightModel,
        Self::flight_loop_callback,
        plugin_state,
      ).expect("Failed to create flight loop");
      flight_loop_cb.schedule_seconds(1.0, true);
      self.flight_loop = Some(flight_loop_cb);
    }

    /// Required function called when the plugin is unloaded by X-Plane.
    fn plugin_stop(&mut self) {
      // Remove the flight loop when the plugin is stopped
      self.flight_loop = None;
    }

    /// Required function called when the plugin is enabled by X-Plane.
    fn plugin_enable(&mut self) {}

    /// Required function called when the plugin is disabled by X-Plane.
    fn plugin_disable(&mut self) {}

    /// Required function called when the plugin receives a message from another plugin.
    fn plugin_receive_message(&mut self, _msg: PluginMessage) {}
}

// Register the plugin with X-Plane using the `x_plugin_start!` macro.
x_plugin_start!(
    DemoPlugin,
    "Demo Plugin in Rust",
    "com.innermarker.demo",
    "A demo plugin example in Rust, demonstrating the use of the xplm-oxide crate."
);
```
