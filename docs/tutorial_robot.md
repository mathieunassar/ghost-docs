---
id: tutorial_robot
title: Create a Simple Robot Simulation
---
This tutorial introduces the features "module lifetime", "observability" and "connection" of the Ghost framework through a practical example: the creation of a robot simulation that provides its position and can be updated with velocity commands.

| Difficulty    | Intermediate                                                 |
| ------------- | ------------------------------------------------------------ |
| Time To Read  | 60 minutes                                                   |
| Prerequisites | [Tutorial: Setup](tutorial_setup.md), [Guide: module](module.md), [Guide: observability](module_observability.md), [Guide: connection](connection.md) |

## Summary

A very simplistic robot simulation is created. In this tutorial, we consider an omnidirectional robot that can drive in x and y directions on a plane. The simulation evaluates the position of the robot in its odometry coordinate system based on the robot's velocity.

The created microservice will provide two execution modes:

- the robot (usage: "connection_grpc_robot robot"), that publishes its pose as a Google Protobuf message of type ghost::examples::protobuf::RobotOdometry;
- a subscriber version that receives the odometry messages (usage: "connection_grpc_robot").

## Tutorial

### Step 1: Preparation of the Project

The example program is located in the template repository: https://github.com/mathieunassar/ghostmodule-template.

As a first step in this tutorial, we clone the repository and build the example. Please refer to the [setup tutorial](tutorial_setup.md) for detailed instructions about how to do that.

### Step 2: the Robot Model

Let's jump into the code! To keep the example minimalist and easy to read, the full program is located in one file: src/connection_grpc_robot.cpp.

####  The main function

In Ghost programs, the main function is responsible for the configuration of the module. We create a ghost::ModuleBuilder, which collects the configured components:

```
auto builder = ghost::ModuleBuilder::create();
```

In our program, we will need to register custom commands and start connections. These operations belong to the initialization phase of the module, so we register a function to the builder to be called during this phase:

```
builder->setInitializeBehavior(std::bind(&RobotModule::initialize, &myModule, std::placeholders::_1));
```

The class RobotModule will represent the runtime state of the program.

After the initialization, the robot program will cyclically update the robot's pose. We will perform this update in a run() method, which we also register to the builder:

```
builder->setRunningBehavior(std::bind(&RobotModule::run, &myModule, std::placeholders::_1));
```

We will also need access to the console to type commands, a logger to print the odometry to the console, and we will parse the program options to determine whether to start the robot (publisher), or a subscriber of the odometry. This is done by adding these configurations to the builder:

```
auto console = builder->setConsole();
builder->setLogger(ghost::GhostLogger::create(console));
builder->setProgramOptions(argc, argv);
```

Finally, we build the module and run it:

```
std::shared_ptr<ghost::Module> module = builder->build("ghostRobotExample");
if (module) module->start();
```

At this point the module starts, the initialization method is called, which is followed by cyclic calls to run() until the module is stopped.

#### The Initialize method

We provided an initialize method to the ghost::ModuleBuilder. It expects a function with the following signature:

`bool initialize(const ghost::Module& module)`

We therefore declare this function within the class RobotModule:

```
bool initialize(const ghost::Module& module)
{
   _connectionManager = ghost::ConnectionManager::create();
   
   if (module.getProgramOptions().hasParameter("__0") &&
     module.getProgramOptions().getParameter<std::string>("__0") == "robot")
   {
     _robot = std::make_shared<Robot>(_connectionManager, _configuration);
   }
   
   return true;
}
```

In this implementation, we look into the program options to check whether to start the robot or a subscriber. If an unnamed parameter is provided in the first position and its value is "robot", then we create a robot. We will deal with the connections between the publisher and subscribers in the step 4.

#### The run method

After the module is initialized, we want to keep updating the robot based on a model that modifies its pose based on its velocity. For that, we registered the RobotModule's run() method to the builder. If the module's initialization succeeds, it will be called cyclically until the module is stopped or the function returns false.

This is an implementation of run():

```
bool run(const ghost::Module& module)
{
	if (_robot) _robot->update();
	// The robot will send new odometry data (roughly) with a 2 Hz frequency
	std::this_thread::sleep_for(std::chrono::milliseconds(500));
	return true;
}
```

The class Robot is responsible for holding the robot model and therefore exposes an update method:

```
void update()
{
    auto delta = std::chrono::steady_clock::now() - _lastTime;
    // Perform very complex operations to compute the super-high-resolution odometry.
    _odoX = _odoX + _velX * std::chrono::duration_cast<std::chrono::milliseconds>(delta).count() / 1000;
    _odoY = _odoY + _velY * std::chrono::duration_cast<std::chrono::milliseconds>(delta).count() / 1000;
    _lastTime += delta;
    std::cout << "new pose: x: " << _odoX << " and y: " << _odoY << std::endl;
}
```

In this example, the run() function always return true. The only way to stop the module (apart from killing it) is to activate the console (by pressing "enter" so that the prompt is displayed) and type in the "exit" command.

### Step 3: a Command to Update the Velocity

After step 2, the robot can be started by starting the executable with the "robot" program option. At this point, the robot only drives with a zero velocity, and therefore does not move.

Let's add a console command to set the velocity of the robot, that we can call from the robot program.

In order to create a command, all we need is to create a class deriving from ghost::Command, implement the virtual methods and register it to the module's instance of ghost::CommandLineInterpreter:

```
class UpdateVelocityCommand : public ghost::Command
{
public:
	UpdateVelocityCommand(const std::shared_ptr<Robot>& robot) : _robot(robot)
	{
	}

	// The execute method corresponds to the action of this command.
	bool execute(const ghost::CommandLine& commandLine, const ghost::CommandExecutionContext& context) override
	{
		if (!commandLine.hasParameter("__0") || !commandLine.hasParameter("__1") || !_robot) return false;

		double vx = commandLine.getParameter<double>("__0");
		double vy = commandLine.getParameter<double>("__1");
		_robot->setVelocity(vx, vy);

		return true;
	}

	std::string getName() const override
	{
		return "UpdateVelocityCommand";
	}
	// This method defines the command that he user will have to enter to invoke this command
	std::string getShortcut() const override
	{
		return "updateVel";
	}
	std::string getDescription() const override
	{
		return "Updates the robot's velocity";
	}

private:
	std::shared_ptr<Robot> _robot;
};
```

The class is straightforward: the `getShortcut()` method specify the text that the user must type in the console to trigger the command. The execute method checks that two unnamed parameters are provided as the first and second parameters of the command, and calls `setVelocity()` on the Robot instance to set the velocity.

In order to register the command, we add the following lines to the initialize method of the RobotModule class:

```
auto command = std::make_shared<UpdateVelocityCommand>(_robot);
module.getInterpreter()->registerCommand(command);
```

You can now try to execute the program and do the following:

- activate the console by pressing "enter" so that the prompt is displayed
- type "updateVel 1.0 2.0", so that the robot moves with a velocity of 1 m/s in the x direction and 2 m/s in the y direction.

### Step 4: the Odometry Publisher and their Subscribers

The final step of this tutorial is to publish the updated pose from the robot program, and to subscribe to this data from the subscriber programs. For this, we use the ghost::ConnectionManager together with Ghost's Google gRPC implementation of the connections.

#### Initialization of the Connections

In order to work with connections, we need a ghost::ConnectionManager instance. We also want to use networked connections between the odometry publisher and the subscribers: this can be easily configured by the extension's entry point class ghost::ConnectionGRPC. We therefore add the two lines to the initialization method:

```
_connectionManager = ghost::ConnectionManager::create();
ghost::ConnectionGRPC::initialize(_connectionManager);
```

The second line adds rules to the connection manager's factory that link all connection configurations of the type ghost::ConnectionConfigurationGRPC to implementations using google gRPC.

We also define that the publisher will be reachable on the localhost on the port 8562:

```
_configuration.setServerIpAddress("127.0.0.1");
_configuration.setServerPortNumber(8562);
```

Based on the connection manager and this configuration, we can create the publisher in the constructor of the Robot class:

```
auto publisher = connectionManager->createPublisher(config);
if (publisher->start())
	_odometryWriter = publisher->getWriter<ghost::examples::protobuf::RobotOdometry>();
```

The last line gets a ghost::Writer object that is used to push messages of type ghost::examples::protobuf::RobotOdometry to the publisher.

We can continue with creating the subscriber:

```
auto subscriber = _connectionManager->createSubscriber(_configuration);

auto messageHandler = subscriber->addMessageHandler();
messageHandler->addHandler<ghost::examples::protobuf::RobotOdometry>(&odmetryMessageHandler);
```

and start it:

```
bool subscriberStartResult = subscriber->start();
if (!subscriberStartResult)
{
	GHOST_ERROR(module.getLogger()) << "Couldn't find the robot! Start this program with the \"robot\" option.";
}
```

You may have noticed that we added a message handler to the subscriber. We also added a handler for messages of the odometry type. The handler "odometryMessageHandler" is a function that processes incoming messages. In our example, we will simply print it:

```
void odmetryMessageHandler(const ghost::examples::protobuf::RobotOdometry& message)
{
	std::cout << "Received odometry: " << message.x() << "; " << message.y() << " [m/s; m/s]" << std::endl;
}
```

#### Publish the Odometry

The last piece of the connection puzzle is... to actually send the odometry. Since we created a ghost::Writer from the publisher in the Robot class, we can simply use it and extend the Robot's update() method with the sending part:

```
auto msg = ghost::examples::protobuf::RobotOdometry::default_instance();
msg.set_x(_odoX);
msg.set_y(_odoY);

_odometryWriter->write(msg);
```

You can now compile the program again and start a "robot" instance, followed by multiple subscriber instances (simply start the program without options). If everything worked well, the subscribers should all receive odometry messages twice per second! You can also test the "updateVel" command by setting a new velocity. Once you do that, the subscribers should start receiving updated poses based on the new velocity.

**Congratulations! You successfully created your first Ghost microservice!** You can now spend some time to investigate the rest of the features and start building your own programs with the Ghost framework.