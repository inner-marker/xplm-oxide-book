# Barebones Plugin

This is the bare bones of an X-Plane plugin using the crate xplm_oxide.

We will cover each part in detail.

```toml
## Cargo.toml

[package]
name = "barebones_plugin"
version = "0.1.0"
edition = "2024"

[lib]
crate-type = ["cdylib"]
```

```rust
// src/lib.rs
use xplm_oxide::*;

#[xplm_plugin]
pub struct BareBonesPlugin;

impl Plugin for BareBonesPlugin {
    fn plugin_start(&mut self) {}
    fn plugin_stop(&mut self) {}
    fn plugin_enable(&mut self) {}
    fn plugin_disable(&mut self) {}
    fn plugin_receive_message(&mut self, _msg: PluginMessage) {}
}

x_plugin_start!(
    BareBonesPlugin,
    "Barebones Plugin in Rust",
    "com.innermarker.barebones",
    "A bare-bones plugin example in Rust, demonstrating the use of the xplm-oxide crate."
);
```

This plugin does not do anything except register itself with X-Plane. It will be displayed in the plugin manager, but it will not have any functionality. We will add functionality in later examples.

## Cargo.toml

First, in `Cargo.toml`, we specify that the crate type is `cdylib`.

X-Plane plugins are just dynamically-linked libraries (DLL) under the hood. The compiled code will not be a stand-alone executable, but rather a library that X-Plane will load at runtime.

The file extensions will depend on the operating system:

- `.dll` for Windows,
- `.dylib` for macOS,
- `.so` for Linux.

In the case of the `barebones_plugin` example above, the compiled library will be named `barebones_plugin.dll`, `barebones_plugin.dylib`, or `barebones_plugin.so` depending on the target platform.

## Building and Installing the Plugin

Once we run `cargo build`, we need to copy the compiled file from `[crate]/target/debug` to the `X-Plane 12/Resources/plugins/[our plugin]/64/`. Then, we rename the file as follows:

- Windows: `barebones_plugin.dll` => `win.xpl`
- macOS: `barebones_plugin.dylib` => `mac.xpl`
- Linux: `barebones_plugin.so` => `lin.xpl`

To summarize, here are the steps:

1. Build the plugin with `cargo build`. (or `cargo build --release` for a release build)
2. Copy the compiled file from `[crate]/target/debug` (or `[crate]/target/release` for a release build) to `X-Plane 12/Resources/plugins/[our plugin]/64/`.
3. Rename the file according to the platform.

There should now be a file named `win.xpl`, `mac.xpl`, or `lin.xpl` in `X-Plane 12/Resources/plugins/[our plugin]/64/`.

---

## Breaking Down lib.rs

```rust
use xplm_oxide::*;
```

In this section, we will discuss what each part of the `lib.rs` file does.

### Importing the xplm_oxide Crate

First, we import the crate `xplm_oxide` and all of its contents.

At preseant, there is no `prelude` module in `xplm_oxide`, so we have to import everything. In the future, a `prelude` module may be added.

---

### Defining the Plugin Struct

```rust
#[xplm_plugin]
pub struct BareBonesPlugin;
```

Next, we define a `struct` that will represent our plugin. All plugins that are built using `xplm_oxide` are just a `struct`.

In this case, we call it `BareBonesPlugin`. You can use any name you like.

If you want to have access to any data during the life of the plugin, create fields here. You will see that in later examples in this book. For now, our barebones plugin does not use any data, so we leave the `struct` empty.

Additionally at this step, we use the `#[xplm_plugin]` attribute macro to mark this `struct` as a plugin. This is required for the plugin to be recognized by X-Plane.

This macro handles a number of tasks under the hood. Primarily, it exposes a few `unsafe extern "C"` functions that X-Plane will call at various points in the plugin's lifecycle.

---

### Implimenting the Plugin Trait

```rust
impl Plugin for BareBonesPlugin {
    fn plugin_start(&mut self) {}
    fn plugin_stop(&mut self) {}
    fn plugin_enable(&mut self) {}
    fn plugin_disable(&mut self) {}
    fn plugin_receive_message(&mut self, _msg: PluginMessage) {}
}
```

Next, we implement the `Plugin` trait for our `BareBonesPlugin` struct.

Note that there are five functions that we must implement. These functions are required by X-Plane.

1. `plugin_start`: This function is called when the plugin is first loaded by X-Plane. It is called only once during the lifetime of the plugin.
2. `plugin_stop`: This function is called when the plugin is unloaded by X-Plane. It is called only once during the lifetime of the plugin.
3. `plugin_enable`: This function is called when the plugin is enabled by X-Plane. It may be called multiple times during the lifetime of the plugin. When an X-Plane user opens the plugin manager in the simulator, they can enable or disable plugins. This function is called when the plugin is enabled.
4. `plugin_disable`: This function is called when the plugin is disabled by X-Plane. It may be called multiple times during the lifetime of the plugin. When an X-Plane user opens the plugin manager in the simulator, they can enable or disable plugins. This function is called when the plugin is disabled.
5. `plugin_receive_message`: This function is called when the plugin receives a message from another plugin. It may be called multiple times during the lifetime of the plugin. The message is passed as a `PluginMessage` struct.

X-Plane does not require any particular functionality in these functions; they can be left empty. However, we will see in later examples that these functions are where we will implement the functionality of our plugin.

---

### The x_plugin_start! Macro

```rust
x_plugin_start!(
    BareBonesPlugin,
    "Barebones Plugin in Rust",
    "com.innermarker.barebones",
    "A bare-bones plugin example in Rust, demonstrating the use of the xplm-oxide crate."
);
```

Finally, we use the `x_plugin_start!` macro to define the plugin's metadata. This macro takes four arguments:

1. The plugin `struct` that we defined earlier.
2. The plugin's name, which will be displayed in the X-Plane plugin manager.
3. The plugin's signature, which is a unique identifier for the plugin. It is recommended to use a reverse URL format for the signature, such as `com.innermarker.barebones`.
4. The plugin's description, which will be displayed in the X-Plane plugin manager.