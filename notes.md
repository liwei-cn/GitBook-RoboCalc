The following are current constraints of usage of RoboChart for constructing a simulation. Some of them are not the constraints of RoboChart language, while the others are for the user to construct a useful simulation.

* The name and type of the event used for connecting communication between different components should be the same.

* Only asynchronous communication is supported.

* No composite state is supported \(TODO\).

* The generated code is not targeted for some specific simulator. The user needs to extend the framework to fit different needs.

* No operation is supported \(the user can specify composite state if needed\).

* The clock should only be defined inside the state machine.



