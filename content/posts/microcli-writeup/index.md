+++
date = '2019-12-01T10:00:00-05:00'
draft = false
title = 'MicroCLI: A Lightweight CLI Framework for Bare Metal Systems'
+++
When working on embedded systems, I typically rely on some form of serial
interface for debugging the firmware and perhaps issuing some control commands.
In my years doing embedded systems development, I haven't come across any great
frameworks for building a more fully-featured command interface.

I'm talking specifically about bare metal projects, perhaps even without a
standard library available. Of course, if your device can run linux, you have
loads of great options for hosting a shell and building a CLI... but there's a
gap for lower level systems. I built
[MicroCLI](https://github.com/phreaknik/microcli) to address this gap by
providing a reusable command-line interpreter framework specifically designed
for resource-constrained embedded systems.

MicroCLI was built to fit into even the most constrained environments:

- **No operating system dependency** - Runs on bare metal systems
- **No standard library requirement** - Works in environments without stdlib
- **Minimal code size** - <1kB overhead
- **No memory allocator necessary** - Can operate in systems without dynamic
memory
- **Limited CPU resources** - Minimal processing overhead

## Key Features

### Flexible I/O Integration

MicroCLI doesn't assume anything about your hardware's I/O capabilities.
Whether you're using UART, USB CDC, or any other serial interface, you provide
the low-level read/write functions, and MicroCLI handles the rest.

### Command Framework

Define your commands with simple function pointers and descriptors. MicroCLI
handles parsing, tokenization, and dispatch. Commands can accept arguments, and
the framework provides argc/argv style access to parameters.

### Default Help Command & Command History

A default 'help' command and command history are provided and enabled by
default, to provide an easy out-of-box experience. These can be disabled and
overridden if the default behavior is not desired.

### Compile-Time Configuration

Enable only what you need. Features like command history, echo, and various
parsing options can be included or excluded at compile time to minimize
footprint.

### Runtime Configurability

Adjust behavior on the fly - change verbosity levels, enable/disable echo, or
modify other settings programmatically based on your system's state.

## Implementation Example

Here's a minimal integration example:

```c
// Define your commands
static int cmd_led(int argc, char *argv[]) {
    if (argc > 1) {
        if (strcmp(argv[1], "on") == 0) {
            LED_ON();
        } else if (strcmp(argv[1], "off") == 0) {
            LED_OFF();
        }
    }
    return 0;
}

// Register commands with MicroCLI
static const cmd_t commands[] = {
    {"led", cmd_led, "Control LED [on/off]"},
    {"status", cmd_status, "Show system status"},
    {NULL, NULL, NULL}
};

// Initialize
microcli_init(commands, uart_putchar);
```

## Technical Details

The core architecture is built around a simple state machine that processes
input characters, builds command lines, and dispatches to registered handlers.
The command table is typically stored in flash/ROM to minimize RAM usage. Input
buffering is static and size-configurable.

When history is enabled, MicroCLI maintains a circular buffer of previous
commands. When disabled, this feature costs zero bytes of RAM.

## Conclusion

MicroCLI provides a practical solution for implementing robust command-line
interfaces in resource-constrained embedded systems. By offering a reusable
framework that respects the limitations of embedded development while providing
modern CLI capabilities, it eliminates the need to repeatedly implement custom
debug interfaces for each project.

MicroCLI is available at
[github.com/phreaknik/microcli](https://github.com/phreaknik/microcli).
Contributions and feedback from the embedded development community are welcome.
