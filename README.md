# Introduction

The framework is formed by a number of C++ classes that implement key features of the RoboChart language. This manual describes how the provided C++ classes are used to build a simulation of a RoboChart model. Currently, the framework supports a limited subset of the RoboChart notation characterised as follows:

* There is a single robotic platform;
* There is a single controller;
* There is a single state machine;
* Entry, exit and transition actions terminate and fit within a single step of the simulation

Despite the constrains of the current simulation framework, it can still be used for modelling a wide variety of robotic controllers. In the next chapter, we demonstrate how these classes are used to construct a simulation of a simple RoboChart model. The final chapter provides more details about the framework classes.

One special aspect that is not enforced by the framework, but must be considered when implementing simulations is the usage of constants, given types \(primitive types\) and simulation specific parameters. Global constants of a model and simulation parameters such as`tstep`\(time step\) can be implemented as class variables and initialised using from an external source to avoid hard-coding and the necessity of recompilation when changes are necessary.

In our simulation targeting the ARGoS platform, these constants and parameters are implemented as variables of the controller class initialised by the module class based on values read from the XML configuration file. This approach relies heavily on ARGoS specific features, and might require some changes in different platforms, but the implementation of constants and parameters as variables in the appropriate classes should be feasible in different platforms.

The treatment of given types \(primitive types\) is slightly less clear because ARGoS have limited configuration mechanisms to define \(reinterpret\) types. The types should probably be hard-coded using C++ type aliasing mechanisms such as`typedef`. For example, a type`ID`could be declared in the simulation code as`typedef int ID`.

The architecture of a simulation is as shown in the picture below, where we consider an example described in the next section as well the general classes of the framework.

![](/assets/simulation-class-diagram.png)

The entry point of the simulation is the execute method of the`Module`class. The blue boxes correspond to framework classes that must be extended to create a simulation. The methods shown in the classes can be implemented in the subclass to provide specific behaviours such as the entry action of a state.

