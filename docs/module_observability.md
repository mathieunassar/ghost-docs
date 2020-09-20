---
id: module_observability
title: Module Observability

---

In this guide, the built-in features of ghostmodule that enable microservice observability are described. The ability to observe the state of a microservice, and possibly interact with it, is an important functionality that can serve multiple purposes such as monitoring, debugging, commissioning or manual operation.

The Ghost framework provides the following components:

- A Logger interface (ghost::Logger);
- the console utility (ghost::Console);
- the command line interpreter (ghost::CommandLineInterpreter);
- a user manager including session management (ghost::UserManager).

## Logging Interface

### Feature Description

A very simple interface logging interface can be set via the ghost::MobuleBuilder during the configuration phase of the microservice. The interface provides five methods to log information at different levels:

- `error()`: feed an error description to the underlying system;
- `warn()`: use this method to indicate that unexpected activity is happening, but no error has occurred yet
- `info()`: prints information about the current operations in the microservice
- `debug()`: writes information that helps analyzing issues and bugs
- `trace()`: most granular type of information, that may also be logged cyclically

The actual implementation is also determined during the configuration phase. ghostmodule exposes two implementations of simple loggers:

- ghost::StdoutLogger: simply prints the content with std::cout with a prefix depending on the information level;
- ghost::GhostLogger: also prints the content to the console, but using the ghost::Console utility, which is described in the next section.

In order to develop specific or more extensive implementations of the ghost::Logger, Users are required to inherit from the interface and implement the corresponding virtual methods.

### Usage

The logger is accessible from the ghost::Module entry point class. In order to write log lines, a few C++ defines exist to make the calls more recognizable and readable. Writing an info log line is done the following way:

`GHOST_INFO(module->getLogger()) << "This is a log line in " << 2020;`

Writing trace, debug, warn and error logs are achieved by replacing the "INFO" letters by the respective matching capital letters.

## Console Utility

### Feature Description

The console utility solves the following problem: how to display log information and expect user input at the same time? The solution implemented by ghost::Console is the following.

The module framework has two console states: Input and Output. While the state is Output, the microservice prints log information. On the contrary, during the Input state, the console saves the output log lines while it expects the User to provide some input. Once the User is finished providing input, the saved log lines are displayed and logging is resumed.

In order to switch from Output to Input, it is simply necessary to press the "Enter" key. A prompt string is then displayed to indicate that the mode is now Input.

##### Activating the Console

Pressing Output again will have the following effect:

- if nothing has been written in the Input mode, then the mode is switched to Output;
- if something has been written in the Input mode, then a command is submitted to the module, and:
  - per default, the mode is switched to Output;
  - if configured, another command is expected by the user.

The following two pictures illustrate an example module in which the mode is Input (top), and in which it is Output (bottom)

![Diagram: ghostmodule and Extensions](assets/ghostmodule_console_input.png)

![Diagram: ghostmodule and Extensions](assets/ghostmodule_console_output.png)

##### Input Modes

During the configuration phase of the microservice, it is possible to set the ghost::InputMode of the console to the following values:

- SEQUENTIAL: once the Input mode is activated, it will only be deactivated by pressing the "Enter" key without writing a command. As a result, Users do not need to switch to the Input mode again in order to provide a sequence of commands.
- DISCRETE: **(Default)** after a command is provided, the console mode is automatically switched back to Output. Hence, in order to write multiple commands, it is necessary to activate the Input mode before each command.

### Usage

If this functionality is necessary in a project, the console must be explicitly activated during the configuration phase of the microservice. For this purpose, the ghost::ModuleBuilder provides the `setConsole()` method. This call sets up a ghost::Console object and returns it, so that it can be configured.

Note: at this point, the console is not active: it will be started during the initialization phase of the ghost::Module. Users may however start the console earlier by calling `start()` on the retrieved console object.

During the configuration of the console, the following parameters can be configured:

- the ghost::InputMode, as mentioned above;
- the format of the prompt, that is displayed when the Input mode is activated
- a callback to call when a new command is provided (per default, the ghost::Module registers a callback to links the user input to the ghost::CommandLineInterpreter)

## Command Line Interpretation

### Feature Description

coming soon...