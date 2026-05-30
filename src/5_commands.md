# Commands

This section will cover working with commands.

This section will follow the example in the xplm-oxide crate called `examples/ex5_three_position_switch.rs`.

You can create new commands and write a function to handle the command. You can also find a command and add functionality to it before X-Plane or other plugins deal with it. You can even entirely override a command.

Commands all have a name that looks like a path. For instance, the command to pause the simulator is called `sim/operation/pause_toggle`. Commands also have a description. In this case, the description for the `pause_toggle` command is:

> Toggle simulation paused state.



## Steps to Create a Command

These are the steps to create a custom command.

1. Define a struct field for an `Option<Command>` and another for an `Option<CommandHandler>`.
2. Initialize both the `Command` and `CommandHandler` to `None` in the `new()` method.
3. In `plugin_start()`, create the new command and save its refrence in the plugin struct
   1. Alternatively, find a command and store its refrence
4. In `plugin_start()` immedielty after the previous step, register the command handler function.
5. In `plugin_stop()`, unregister the command handler and destroy the command by setting both fields to `None`
6. Impliment a the command handler function.

Let's examine each of those steps in more detail.

## Example Walk-Through

We will write a plugin that impliments the three-position battery/alternator switches that some airplanes have. The switch has three positons: Off, Battery, and Battery+Alternator.

We will create two commands:

1. Switch up one position
2. Switch down one position

### Declare the Struct Fields.

Whether you are creating a new command or finding an existing one, you will need to declare struct fields to hold the `Command` and `CommandHandler`.

```rust
#[xplm_plugin]
struct ThreePositionSwitchPlugin {
    // Datarefs here

    // Commands and handlers
    switch_up_cmd: Option<Command>,
    switch_down_cmd: Option<Command>,

    switch_up_cmd_handler: Option<CommandHandler>,
    switch_down_cmd_handler: Option<CommandHandler>,

    // Flight Loop here

    // ...
}
```

### Initialize the Fields

All fields should be initialized to `None` in the `new()` method.

```rust
impl ThreePositionSwitchPlugin {
    pub fn new() -> Self {
        Self {
            // Datarefs here

            // Commands and handlers
            switch_up_cmd: None,
            switch_down_cmd: None,
            switch_up_cmd_handler: None,
            switch_down_cmd_handler: None,

            // Flight Loop here
        }
    }

    // ...
}
```

### Create Commands

In the `plugin_start()` funciton, we create the commands. We call `Command::crerate(...)`.  We pass in the two things we discussed earlier: the name and description. Note that `create()` returns a `Result<Command, CommandError>` that we need to handle.

In the `Err` case, we are using another feature of this crate to log a message to the X-Plane `Log.txt` and a plugin-specific log file. More on logging will be said in another section of this book.

```rust
// in plugin_start()
self.switch_down_cmd = match Command::create(
    "example_plugin/switches/master_switch_down", // name
    "Move master switch down", // description
) {
    Ok(cmd) => Some(cmd),
    Err(err) => {
        log_to(LogDestination::Both, format!("three_position_switch: failed to create switch down command: {err}\n"));
        None
    }
};
```

### Register Command Handler the Function

A command is nothing if it does nothing. Therefore, we need to tell X-Plane what function to call in our plugin to handle the command when it happens.

The command handler registration should follow immediately after creating the command within the `plugin_start()` function.

Because the __command__ is stored as an `Option<Command>`, we need to use `if let` to safely access it. We can use the pattern below to register the struct method `Self::switch_down_command_handler()` as the callback function.

The function `register_plugin_handler()` takes three arguements:

- `before_xplane: bool` - If true, X-Plane calls our handler before its own handler for this command. This rarely matters for custom commands.
- `handler: fn(&mut Self, CommandRef, CommandPhase) -> CommandResult` - The function to call when the command is triggered. It must have the signature shown here.
- `state_accessor: plugin_state` - A mutable reference to the plugin state, allowing the handler to modify the plugin's data. This is a value created when we invoke `#[xplm_plugin]` on our plugin struct. We need to pass it to command handlers if we intend to work with any datarefs or other plugin struct fields.

```rust
if let Some(switch_down_cmd) = &self.switch_down_cmd {
    self.switch_down_cmd_handler = Some(switch_down_cmd.register_plugin_handler(
        true,
        Self::switch_down_command_handler,
        plugin_state,
    ));
}
```

### Unregister Command Handler

When the plugin stops, we should unregister the command handlers to clean up properly. This is done in the `plugin_stop()` function. 

```rust
fn plugin_stop(&mut self) {
  self.switch_up_cmd = None;
  self.switch_down_cmd = None;
    self.switch_up_cmd_handler = None;
    self.switch_down_cmd_handler = None;
}
```

### Write the Command Handler

When we impliment our plugin struct, we can write command handler functions. The function signature must look like:

```rust
fn switch_up_command_handler(&mut self, phase: CommandPhase) -> CommandHandlerResult {
```

The `phase` arguement is important. It tells us whether a command is just begining to be executed, continuing to be executed from the previous frame, or finishing.

For instance, You may want to fire a missile the instant a pilot presses the trigger: that would execute on `CommandPhase::Begin`. You may want to keep re-executing a task while the pilot holds a button; here you use `CommandPhase::COntinue`. Finally, you may want to execute some other function only when the pilot releases the button: `CommandPhase::End`.  You could use `if` or `match` statements to handle the cases.

The final thing to do in your command handler is to return a `CommandHandlerResult`. This is an `Enum` that tells X-Plane whether to continue processing the command or not.

- `CommandHandlerResult::Continue` - Let X-Plane and other plugins continue processing the command.
- `CommandHandlerResult::Consume` - Stop X-Plane and other plugins from processing the command. This is useful if you want to override a command.

Here is a cut-down example of the command handler function for the switch_up command.

```rust
fn switch_up_command_handler(&mut self, phase: CommandPhase) -> CommandHandlerResult {

  //get dataref values you will need here

  match phase {
    CoommandPhase::Begin => {
      // Do something when the command begins
    },
    CommandPhase::Continue => {
      // Do something while the command is being held
    },
    CommandPhase::End => {
      // Do something when the command ends
    }
  }

  // set dataref values here

  // Always return a `CommandHandlerResult`
  CommandHandlerResult::Continue
}
```

## Example Full Code

`ex5_three_position_switch.rs`

```rust
//! This example combines multiple concepts to impliment a three-position switch.
//! 
//! Some Cessna airplanes have a three-position master switch. The positions are:
//! - Off: battery and alternator are off
//! - Battery: battery is on, alternator is off
//! - Both: battery and alternator are both on
//! 
//! In order to simulate this functiuonality in the plugin and the 3D model, we will need a few things.
//! 1. a dataref to track the switch positon. Blender visual animations will use this dataref.
//! 2. Write access to he battery and alternator (generator) datarefs from X-Plane. As the pilot moves
//!    the switch, we will update the battery and alternator datarefs to match the switch position.
//! 3. Commands to move the switch up or down one position. A designer in Blender would attach these commands
//!    to a manipulator in the 3D `cockpit.obj` file.
//! 
//! Note that the handlers for the two commands change only the value of our custom dataref
//! for our three-position switch. In the flight loop is where we update the values for
//! the battery (DataRefInt) and the alternator (DataRefIntArray) based on the value of the 
//! custom switch position dataref.
//! 
//! The alternator dataref is called "generator_on" because it is used for both generators and 
//! alternators in X-Plane. Also, the generator dataref refers to an integer array. We only care about 
//! Alternator #1, which is the first element of that array. Setting that element to 1 turns the 
//! alternator on, and setting it to 0 turns the alternator off.

use xplm_oxide::*;

#[xplm_plugin]
struct ThreePositionSwitchPlugin {
    // Datarefs
    battery_on_dr: Option<DataRefInt>,
    alternator_on_dr: Option<DataRefIntArray>,
    switch_position_dr: Option<DataRefInt>,

    // Commands and handlers
    switch_up_cmd: Option<Command>,
    switch_down_cmd: Option<Command>,

    switch_up_cmd_handler: Option<CommandHandler>,
    switch_down_cmd_handler: Option<CommandHandler>,

    // Flight Loop
    flight_loop: Option<FlightLoop>,
}

impl ThreePositionSwitchPlugin {
    pub fn new() -> Self {
        Self {
            // Datarefs
            battery_on_dr: None,
            alternator_on_dr: None,
            switch_position_dr: None,

            // Commands and handlers
            switch_up_cmd: None,
            switch_down_cmd: None,
            switch_up_cmd_handler: None,
            switch_down_cmd_handler: None,

            // Flight Loop
            flight_loop: None,
        }
    }

    /// Switch up command handler.
    ///
    /// This moves the switch up one position, unless it's already in the top position.
    fn switch_up_command_handler(&mut self, phase: CommandPhase) -> CommandHandlerResult {
        
        // A good practice, dependng on what you are simulating, is to only handle commands on the "End" phase. 
        // This way, if a command is held down, you only get one response when the command starts, and one 
        // response when the command ends, rather than multiple responses as the command is held down.
        if phase == CommandPhase::End {
            // get the value for our custom switch position dataref. If it's unavailable, log a message and return.
            let Some(switch_position_dr) = &self.switch_position_dr else {
                log_msg("three_position_switch: switch up command handler: switch position dataref unavailable\n".to_string());
                return CommandHandlerResult::Continue;
            };
            let current_position = switch_position_dr.get();

            // Adjust the switch position up or log a message if we are already in the top position. 
            if current_position < 2 {
                let new_position = current_position + 1;

                // update the dataref value
                let _ = switch_position_dr.set(new_position);
                log_to(LogDestination::Both, format!("three_position_switch: switch moved up to position {new_position}\n"));
            } else {
                log_to(LogDestination::Both, "three_position_switch: switch is already in top position; cannot move up\n".to_string());
            }
        }

        CommandHandlerResult::Continue
    }

    /// Switch down command handler.
    ///
    /// This moves the switch down one position, unless it's already in the bottom position.
    fn switch_down_command_handler(&mut self, phase: CommandPhase) -> CommandHandlerResult {
        if phase == CommandPhase::End {
            // If the dataref is Some(...), get the dataref handle. If the dataref is None, log a message and return.
            let Some(switch_position_dr) = &self.switch_position_dr else {
                log_msg("three_position_switch: switch down command handler: switch position dataref unavailable\n".to_string());
                return CommandHandlerResult::Continue;
            };

            // get the switch position value
            let current_position = switch_position_dr.get();
            if current_position > 0 {
                let new_position = current_position - 1;

                // update the dataref value
                let _ = switch_position_dr.set(new_position);
                log_to(LogDestination::Both, format!("three_position_switch: switch moved down to position {new_position}\n"));
            } else {
                log_to(LogDestination::Both, "three_position_switch: switch is already in bottom position; cannot move down\n".to_string());
            }
        }

        CommandHandlerResult::Continue
    }

    /// Flight loop callback.
     fn flight_loop_handler(&mut self, _sec_last_call: f32, _sec_last_fl: f32, _counter: i32) -> f32 {
         // Get the current value of our custom switch position dataref. If it's unavailable, 
         // log a message and return.
         let switch_pos_val = match &self.switch_position_dr {
             Some(dr) => dr.get(),
             None => {
                 log_msg("three_position_switch: flight loop handler: switch position dataref unavailable\n".to_string());
                 return 1.0; // try again in 1 second
            }
        };
            
        // Depending on the switch position, set the battery and alternator datarefs accordingly.
        if switch_pos_val == 0 {
            log_to!(LogDestination::Both, "three_position_switch: switch in OFF position\n");
            // Off position: battery and alternator off
            if let Some(battery_on_dr) = &self.battery_on_dr {
                let _ = battery_on_dr.set(0);
            }

            // Here, we are setting the first element (position 0) of an integer array dataref.
            if let Some(alternator_on_dr) = &self.alternator_on_dr {
                let _ = alternator_on_dr.set(&[0], 0);
            }
        } else if switch_pos_val == 1 {
            log_to!(LogDestination::Both, "three_position_switch: switch in BATTERY position\n");
            // Battery position: battery on, alternator off
            if let Some(battery_on_dr) = &self.battery_on_dr {
                let _ = battery_on_dr.set(1);
            }
            if let Some(alternator_on_dr) = &self.alternator_on_dr {
                let _ = alternator_on_dr.set(&[0], 0);
            }
        } else if switch_pos_val == 2 {
            log_to!(LogDestination::Both, "three_position_switch: switch in BOTH position\n");
            // Both position: battery and alternator on
            if let Some(battery_on_dr) = &self.battery_on_dr {
                let _ = battery_on_dr.set(1);
            }
            if let Some(alternator_on_dr) = &self.alternator_on_dr {
                let _ = alternator_on_dr.set(&[1], 0);
            }
        } else {
            log_to(LogDestination::Both, format!("three_position_switch: flight loop handler: invalid switch position value {switch_pos_val}\n"));
        }
            
        -1.0 // schedule next frame
     }
}

impl Plugin for ThreePositionSwitchPlugin {
    fn plugin_start(&mut self) {
        // Find the battery and alternator datarefs from X-Plane.
        self.battery_on_dr = match DataRefInt::find("sim/cockpit/electrical/battery_on") {
            Ok(dr) => Some(dr),
            Err(err) => {
                log_to(LogDestination::Both, format!("three_position_switch: failed to find battery dataref: {err}\n"));
                None
            }
        };

        // Note: there is not dataref for alternators. X-Plane uses the same dataref 
        // for both generators and alternators
        self.alternator_on_dr = match DataRefIntArray::find("sim/cockpit/electrical/generator_on") {
            Ok(dr) => Some(dr),
            Err(err) => {
                log_to(LogDestination::Both, format!("three_position_switch: failed to find alternator dataref: {err}\n"));
                None
            }
        };

        // Create our custom dataref to track the switch position.
        self.switch_position_dr = match DataRefInt::create(
            "example_plugin/switches/master_switch_position",
            0,
        ) {
            Ok(dr) => Some(dr),
            Err(err) => {
                log_to(LogDestination::Both, format!("three_position_switch: failed to create switch position dataref: {err}\n"));
                None
            }
        };

        // Create commands and handlers for moving the switch up and down.
        self.switch_down_cmd = match Command::create(
            "example_plugin/switches/master_switch_down",
            "Move master switch down",
        ) {
            Ok(cmd) => Some(cmd),
            Err(err) => {
                log_to(LogDestination::Both, format!("three_position_switch: failed to create switch down command: {err}\n"));
                None
            }
        };
        // register handler
        if let Some(switch_down_cmd) = &self.switch_down_cmd {
            self.switch_down_cmd_handler = Some(switch_down_cmd.register_plugin_handler(
                true,
                Self::switch_down_command_handler,
                plugin_state,
            ));
        }

        // Create a new commsnd
        self.switch_up_cmd = match Command::create(
            "example_plugin/switches/master_switch_up",
            "Move master switch up",
        ) {
            Ok(cmd) => Some(cmd),
            Err(err) => {
                log_to(LogDestination::Both, format!("three_position_switch: failed to create switch up command: {err}\n"));
                None
            }
        };
        // register handler
        if let Some(switch_up_cmd) = &self.switch_up_cmd {
            self.switch_up_cmd_handler = Some(switch_up_cmd.register_plugin_handler(
                true,
                Self::switch_up_command_handler,
                plugin_state,
            ));
        }

        // Create a flight loop that has access to the plugin state.
        let flight_loop_callback = FlightLoop::create_plugin_fn(
            FlightLoopPhase::AfterFlightModel,
            Self::flight_loop_handler,
            plugin_state,
        )
        .expect("Failed to create 5-second flight loop");

        // schedule the first call to happen in 5 seconds.
        flight_loop_callback.schedule_every_frame();

        // Store the flight loop handle in the plugin's state
        self.flight_loop = Some(flight_loop_callback);


    }

    fn plugin_stop(&mut self) {
        self.flight_loop = None;
        self.switch_up_cmd_handler = None;
        self.switch_down_cmd_handler = None;
        self.switch_up_cmd = None;
        self.switch_down_cmd = None;
        self.switch_position_dr = None;
        self.battery_on_dr = None;
        self.alternator_on_dr = None;
    }

    fn plugin_enable(&mut self) {}

    fn plugin_disable(&mut self) {}

    fn plugin_receive_message(&mut self, msg: xplm_oxide::PluginMessage) {
        log_to(LogDestination::Both, format!("three_position_switch: received message:\n{:?}\n", msg));
    }
}

x_plugin_start!(
    ThreePositionSwitchPlugin::new(),
    "Three Position Switch Example",
    "com.innermarker.xplm-oxide.three_position_switch_example",
    "An example plugin that demonstrates a three position switch using datarefs, commands, and a flight loop."
);
```
