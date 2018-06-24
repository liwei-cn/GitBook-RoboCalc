## HardwareComponent.h {#hardwarecomponenth}

A subclass of`HardwareComponent`should implement two methods`Sensors()`and`Actuators()`. These classes provide a connection between the simulator \(ARGoS in our case\) and the RoboChart model. An extension of `HardwareComponent`function is as follows:

```cpp
//HardwareComponent.h
/***********epuck*************/
#include <argos3/plugins/robots/e-puck/control_interface/ci_epuck_wheels_actuator.h>
#include <argos3/plugins/robots/e-puck/control_interface/ci_epuck_base_leds_actuator.h>
#include <argos3/plugins/robots/e-puck/control_interface/ci_epuck_rgb_leds_actuator.h>
#include <argos3/plugins/robots/e-puck/control_interface/ci_epuck_proximity_sensor.h>
#include <argos3/plugins/robots/e-puck/control_interface/ci_epuck_light_sensor.h>

using namespace argos;

class HardwareComponent {

public:
    HardwareComponent() {}
    virtual ~HardwareComponent() {}
    virtual void Sensors() = 0;
    virtual void Actuators() = 0;

    // Sensors
    CCI_EPuckProximitySensor* proximity_sensor_epuck;
    CCI_EPuckLightSensor* light_sensor_epuck;

    // Actuators
    CCI_EPuckWheelsActuator* wheels_actuator;
    CCI_EPuckBaseLEDsActuator* base_leds_actuator;
};
```

In this function, the model \(sensors and actuators\) of the robot \(e-puck in our case\) is defined.

## Interface {#hardwarecomponenth}

The interface class implements the `HardwareComponent` class. For a particular implementation, the `Sensors` function needs to be expanded to check whether the events happen or not. In our example, it is `obstacle`.

# Robotic Platform {#robotic-platform}

The hardware specific operations, variables and events of the robotic platform can be either grouped in interfaces or defined inside the platform. If the operations and/or variables are grouped in interfaces, a robotic platform should _provides \(represented by a symbol **P **in the RoboChart diagram_\) these interfaces_. These operations grouped in the interfaces can then be used later in the controller or state machine. In order to use the operations, the controller/state machine need to require \(represented by a symbol **R **in the RoboChart diagram\) the interfaces provided by a robotic platform. Therefore, in the simulation, the controller/state machine class needs to have a reference of the robotic platform class. _

_A robotic platform also uses \(represented by a symbol **i**_ in the diagram_\) interfaces that group events. Note that the operations and events can not grouped in the same interface. The events are connected between different components to exchange message._ The implementation of a robotic platform must extend Interface subclasses.

