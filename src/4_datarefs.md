# Datarefs

Datarefs are the way that plugin authors read and write the data values to X-Plane.

FOr instance, a plugin developer writing a plugin simulating an altimeter might want to know the elevation of the aircraft. The elevation value is available from the simulator.

X-Plane names datarefs in a way resembling a file path. The elevation dataref is names `sim/flightmodel/position/elevation`. This dataref represents a double-precision floating point number which in Rust is an f64.

As authors, we only use the path intially to get a _handle_ to the dataref. The _handle_ is a non-human readable (gibberish) value that out plugin and X-Plane understand that we use to read and write values. We will never need to know the _handles_ exact value; we use struct fields with human-readable names to store the _handle_, and functions to interact with the dataref.

## Steps to Use a Dataref

1. Declare a struct field as `Option<Dataref[type]>`
2. Initialize the struct field as `None` in the `new()` function
3. In `plugin_start()`, call `find()` or `create()` to get a _handle_ to the dataref and store it in the struct field.
4. When the value is needed in a flight loop callback, call `get()`

## Example

In this example, we will read the elevation dataref each frame, convert it to feet, then provide another dataref with the value in feet.

### Declare a Struct Field

This step looks very similar to the flight loop example. We declare a struct field to hold the dataref _handle_.

One additional step here is that we need to specify the data type. Look at [this page](https://developer.x-plane.com/datarefs/) on the X-Plane Developer website to see the data type (and other details) for all X-Plane-provided datarefs.

On the site, we see that `sim/flightmodel/position/elevation` is a `double` (`f64` in Rust). We also see that it is read-only.

Let's declare the field for our dataref.

```rust
#[xplm_plugin]
pub struct FlightLoopWithDatarefsExample {
    // We will need a flight loop
    flight_loop: Option<FlightLoop>,

    // Datarefs are declared as fields of our plugin struct.
    elevation_meters_dr: Option<DataRefDouble>,

    // Here is a dataref that we will create/register outselves
    // I am using th style `_dr` suffix to indicate that this field is a dataref handle,
    // but this is just a convention and not required by xplm-oxide.
    elevation_feet_dr: Option<DataRefDouble>,
}
```

### Initialize the Struct Field

We will initialise all of the struct fields (flight loop and datarefs) to `None` in the `new()` function. We will set them up later in the `plugin_start()` function.

```rust
impl FlightLoopWithDatarefsExample {
    fn new() -> Self {
        Self {

            // In `new()`, we initialize our flight loop fields to `None`. We 
            // will populate them in `plugin_start()`.
            flight_loop: None,

            // In `new()`, we initialize our dataref fields to `None`. We 
            // will populate them in `plugin_start()`.
            elevation_meters_dr: None,
            elevation_feet_dr: None,
        }
    }
}
```

### Find or Create the Datarefs

When we `find()` a dataref, we are asking X-Plane to give us a _handle_ to the dataref. We will use the _handle_ to read the value of the dataref in our flight loop callback. The `find()` function returns a `Result` type, so we will use a `match` statement to handle the `Ok` and `Err` cases.

```rust
fn plugin_start(&mut self) {
  // ... other code here

  // Find the dataref and store the handle in the plugin's state.
  self.elevation_meters_dr = match DataRefDouble::find("sim/flightmodel/position/elevation") {
      Ok(dr) => Some(dr),
      Err(err) => {
          log_msg(format!("flight_loop_with_datarefs: failed to bind sim elevation dataref: {err}\n"));
          None
      }
  };

  // ... other code here
}
```

When we `create()` a dataref, we need to specify a few more details.

The `create()` function takes the following arguements:

- `name: &str` - The name of the dataref. This is the name that other plugins will use to find our dataref.
- `initial_value: f64` - The initial value of the dataref.

As with `find()`, the `create()` function returns a `Result` type, so we will use a `match` statement to handle the `Ok` and `Err` cases.

In this example, if there is an error, we set the dataref value to `None` and log a message to the X-Plane log file. We will not be able to use the dataref in our flight loop callback if it is `None`.

```rust
fn plugin_start(&mut self) {
  // ... other code here

  // register a new dataref for elevation in feet, and set it to the current elevation in feet.
  self.elevation_feet_dr = match DataRefDouble::create( "innermarker/flight_loop_example/elevation_feet", 0.0) {
    Ok(dr) => Some(dr),
    Err(err) => {
      log_msg(format!("flight_loop_with_datarefs: failed to create elevation_feet dataref: {err}\n"));
      None
    }
  };
  // ... other code here
}
```

### Read the Dataref Value in the Flight Loop Callback

We will read and write values to our two datarefs in the flight loop callback, a method in the plugin struct.

First, it is a good idea to check to see whether the datarefs are `Some` before attempting to read or write their values. Also, the struct field is currently an `Option<DataRefDouble>`, so we need to use pattern matching to get the inner dataref handle, a `DataRefDouble`.

In this snipped, elevation_meters_dr is an `Option<DataRefDouble>`. `elevation_meters_dr_h` is a `&DataRefDouble`. We will use `elevation_meters_dr_h` to read the value of the dataref.

```rust
let Some(elevation_meters_dr_h) = &self.elevation_meters_dr else {
  log_msg("flight_loop_with_datarefs: elevation dataref unavailable; skipping callback\n".to_string());
  return -1.0;
};
```

Next, we get the value of the dataref.

```rust
let elevation_meters_val: f64 = elevation_meters_dr_h.get();
```

For this example, lets convert meters to feet.

```rust
let elev_ft_val: f64 = elevation_meters_val * 3.28084;
```

Finally, lets set the value of the elevation_feet dataref to the converted value. Again, elev_feet_dr is an `Option<DataRefDouble>`, so we need to use pattern matching to get the inner dataref handle, a `&DataRefDouble`. Also, the `set()` function returns a `Result<(), DataRefWriteError>`. We could handle the `Err` case, but for this example, we will just ignore it by using `_`.

```rust
if let Some(elev_feet_dr) = &self.elevation_feet_dr {
  let _ = elev_feet_dr.set(elev_ft_val);
}
```

## Full Example

Now, here is the full example. This is the code for an example in the xplm-oxide crate, located in `xplm-oxide/examples/ex3_flight_loop_with_datarefs.rs`.

```rust
//! Example plugin that demonstrates flight loops plus dataref creation,
//! lookup, read, and write operations.
//! 
//! This plugin finds X-Planes dataref for elevation in meters. Also, it 
//! creates a new dataref for elevation in feet. Every five seconds, the 
//! plugin reads the elevation in meters, converts it to feet, writes the
//! converted value to the new dataref, and logs (to Log.txt) the elevation in feet.

use xplm_oxide::*;

#[xplm_plugin]
pub struct FlightLoopWithDatarefsExample {
    // We want a flight loop. This will let us do some processing every so often
    // durring X-Plane's run. We can have the flight loop run every so many frames, 
    // every so many seconds, or not at all.
    flight_loop: Option<FlightLoop>,

    // Datarefs are declared as fields of our plugin struct.
    elevation_meters_dr: Option<DataRefDouble>,

    // Here is a dataref that we will create/register outselves
    // I am using th style `_dr` suffix to indicate that this field is a dataref handle,
    // but this is just a convention and not required by xplm-oxide.
    elevation_feet_dr: Option<DataRefDouble>,


}

impl FlightLoopWithDatarefsExample {
    /// Called every 5 seconds by X-Plane.
    /// 
    /// This callback has direct access to &mut self, allowing it to use cached
    /// dataref handles stored in plugin state without incurring the overhead of
    /// calling DataRef::find() on each invocation.
    fn flight_loop_callback_handler(&mut self, _elapsed_since_last_call: f32, _elapsed_since_any_loop: f32, _counter: i32) -> f32 {
        let Some(elevation_meters_dr_h) = &self.elevation_meters_dr else {
            log_msg("flight_loop_with_datarefs: elevation dataref unavailable; skipping callback\n".to_string());
            return -1.0;
        };

        let elevation_meters_val: f64 = elevation_meters_dr_h.get();
    
        // The elevation dataref is in meters. Lets convert it to feet and log it.
        let elev_ft_val: f64 = elevation_meters_val * 3.28084;
    
        // Write the elevation in feet to our custom dataref.
        if let Some(elev_feet_dr) = &self.elevation_feet_dr {
            let _ = elev_feet_dr.set(elev_ft_val);
        }
    
        log_msg(format!("[5-second] callback called, elevation: {} ft\n",elev_ft_val));
    
        -1.0
    }


    fn new() -> Self {
        Self {

            // In `new()`, we initialize our flight loop fields to `None`. We 
            // will populate them in `plugin_start()`.
            flight_loop: None,

            // In `new()`, we initialize our dataref fields to `None`. We 
            // will populate them in `plugin_start()`.
            elevation_meters_dr: None,
            elevation_feet_dr: None,
        }
    }
}

impl Plugin for FlightLoopWithDatarefsExample {
    
    fn plugin_start(&mut self) {
        // Find the dataref and store the handle in the plugin's state.
        self.elevation_meters_dr = match DataRefDouble::find("sim/flightmodel/position/elevation") {
            Ok(dr) => Some(dr),
            Err(err) => {
                log_msg(format!("flight_loop_with_datarefs: failed to bind sim elevation dataref: {err}\n"));
                None
            }
        };
        
        // register a new dataref for elevation in feet, and set it to the current 
        // elevation in feet.
        self.elevation_feet_dr = match DataRefDouble::create(
            "innermarker/flight_loop_example/elevation_feet", // name
            0.0 // initial value
        ) {
            Ok(dr) => Some(dr),
            Err(err) => {
                log_msg(format!("flight_loop_with_datarefs: failed to create elevation_feet dataref: {err}\n"));
                None
            }
        };

        // Create a flight loop that has access to the plugin state.
        let flight_loop_callback = FlightLoop::create_plugin_fn(
            FlightLoopPhase::AfterFlightModel,
            Self::flight_loop_callback_handler,
            plugin_state,
        )
        .expect("Failed to create 5-second flight loop");

        // schedule the first call to happen in 5 seconds.
        flight_loop_callback.schedule_seconds(5.0, true);

        // Store the flight loop handle in the plugin's state
        self.flight_loop = Some(flight_loop_callback);
    }

    fn plugin_stop(&mut self) {
        self.flight_loop = None;
    }

    fn plugin_enable(&mut self) {}
    fn plugin_disable(&mut self) {}
    fn plugin_receive_message(&mut self, _msg: PluginMessage) {}
}

x_plugin_start!(
    FlightLoopWithDatarefsExample::new(),
    "Flight Loop with Datarefs Example in Rust",
    "com.innermarker.xplm-oxide.flight_loop_with_datarefs_example",
    "A flight loop example that demonstrates creating, finding, reading, and writing datarefs."
);
```
