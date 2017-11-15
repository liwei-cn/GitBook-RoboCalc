The following are current constraints of usage of RoboChart for constructing a simulation. Some of them are not the constraints of RoboChart language, while the others are for the user to construct a useful simulation.

* The name and type of the event used for connecting communication between different components should be the same.

* Only asynchronous communication is supported \(TODO\).

* No composite state is supported \(TODO\).

* The generated code is not targeted for some specific simulator. The user needs to extend the framework to fit different needs.

* No operation is supported \(the user can specify composite state if needed\).

* The clock should only be defined inside the state machine.

* The simulation framework can execute multiple transitions at one control cycle, but it may cause the 'infinite loop' issue \(TODO\).

* Defined interfaces only contain events.

* No event trigger is allowed for the ingoing transition of a junction or probabilistic junction.

* A module should be self-contained. The operations used by state machine should be provided by the robotic platform or defined inside the controller.

* There should be be only one entry/exit action in each state, but the entry/exit action can include a sequence of statements.

* The name of variables and operations should be unique.



