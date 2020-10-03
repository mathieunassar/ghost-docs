---
id: tutorial_setup
title: Set Up a New Project Using Ghostmodule
---
In this tutorial you will learn to set up a new repository from a template pre-configured to use the Ghost framework.

| Difficulty    | Beginner                             |
| ------------- | ------------------------------------ |
| Time To Read  | 30 minutes                           |
| Prerequisites | [Guide: ghostmodule](ghostmodule.md) |


## Summary

The Github template (https://github.com/mathieunassar/ghostmodule-template) is used that contains a Conan file configured to resolve the necessary dependencies. The template project contains an example program that is compiled and executed along the tutorial.

## Tutorial

### Step 1: Preparation of the Workspace

Let's begin with some prerequisites. In this tutorial, we will use CMake to configure our example project, and the dependency manager Conan to use the Ghost framework. Make sure that CMake is installed (the minimum required version is 3.8) and proceed to the installation of Conan, if it is not already on your machine. You can find installation instructions here:

- CMake: https://cmake.org/install/
- Conan: https://docs.conan.io/en/latest/installation.html

### Step 2: Cloning the Template Repository

Now your workspace is ready, we can move to the next step and clone the template repository on the following URL: https://github.com/mathieunassar/ghostmodule-template.git.

While git is working hard on cloning the repository, here is a bit of information concerning its content. The template contains:

- a **CMake structure** used to build a very simple example of a program using ghostmodule
- a file named **conanfile.txt**, which contains a list of dependencies for this example project. Among a few examples, you can see `ghostmodule/x.y@mathieunassar/stable`
- some optional files that only brings realism into this project

Unless your Internet connection is very slow, you should now have the template repository locally. Once it's there, we can continue with the next step.

### Step 3: Building the project

Building the project is straightforward, but we must first tell Conan where to look for ghostmodule. This is done by typing the following line into your favorite console:

```
conan remote add ghostrobotics "https://api.bintray.com/conan/mathieunassar/ghostrobotics"
```

And finally:

```
mkdir build
cd build
cmake ..
cmake --build .
```

During the CMake configuration, you should be able to read about Conan reading the local configuration, downloading the package and optionally build it. The build step compiles the example and should also run without problems.

### Step 4: Hello World!

Now navigate to the build folder, where the example executable should now be built, and execute it.

In order to exit the program, trigger the console input mode by pressing the "enter" key, type "exit" and press "enter" again.

**Congratulations! You successfully set up your first Ghost project!** You can now spend some time to investigate into the repository, and have a look at the example code. We will work together on its content in another tutorial.