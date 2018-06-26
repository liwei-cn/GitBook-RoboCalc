The following are current constraints of usage of RoboChart for constructing a simulation. Some of them are the constraints of RoboChart language, while the others are for the user to construct a useful simulation.

* The name and type of the event used for connecting communication between different components should be the same.

* The generated code is not targeted for some specific simulator. The user needs to extend the framework to fit different needs.

* The clock should only be defined inside the state machine.

* The simulation framework can execute multiple transitions at one control cycle, but it may cause the 'infinite loop' issue. This is known as deadlock.

* Defined interfaces only contain events.

* No event trigger is allowed for the outgoing transition of a junction or probabilistic junction.

* A module should be self-contained. The operations used by state machine should be provided by the robotic platform or defined inside the controller.

* There should be be only one entry/exit action in each state, but the entry/exit action can include a sequence of statements.

* The name of variables and operations should be unique.

* Only support since for the clock related condition.

A video showing how to build a basic state machine controller can be found here:

[https://youtu.be/I5L\_PGstu40](https://youtu.be/I5L_PGstu40)

The complete code for the obstacle avoidance example can be found here:

https://drive.google.com/file/d/1xLGi0AyrETHuEqb5r4m4SmGwW8ymMWeG/view?usp=sharing

