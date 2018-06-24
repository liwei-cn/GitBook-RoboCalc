```
Module
```

The module class is the root of the simulation. When used in the ARGoS simulator it implements a`CCI_Controller`class, which provides a`ControlStep()`function that does not take any parameters and describes a single step of execution. Any subclass of`Module`must implement the methods`Init`and `ControlStep`.

In ARGoS, `Init`has the following signature:

```cpp
virtual void Init(argos::TConfigurationNode& t_node);
```

It takes a`TConfigurationNode`that can be used to read parameters from the XML experiment file. The module class overwrites the `ControlStep()` function of the `CCI_Controller`class and calls the `Execute()`function. For the function `Execute()`, in the current version of the framework, it first calls the`Sensors()`method of the robot, then the`Execute()`method of the controller and finally the`Actuators()`method of the robot. The implementation of the module class for the obstacle avoidance example is as follows:

```cpp
//OAModule.h
#include "Robot.h"
#include "ContMovement.h"
#include "channel_types.h"

#include <argos3/core/control_interface/ci_controller.h>

class OAModule : public argos::CCI_Controller {
public:
    OAModule() :
            OAModule_Robot(nullptr),
            OAModule_ContMovement(nullptr),
            obstacle(std::make_shared<robochart::obstacle_channel>("obstacle")) {}
    virtual ~OAModule() {};

    virtual void Init(argos::TConfigurationNode& t_node);
    virtual void ControlStep();
    void Execute();

private:
    std::shared_ptr<robochart::obstacle_channel> obstacle;
    std::shared_ptr<Robot> OAModule_Robot;
    std::shared_ptr<ContMovement> OAModule_ContMovement;
};
```

```cpp
//OAModule.cpp
#include "OAModule.h"
#include "StmMovement.h"

void OAModule::Init(argos::TConfigurationNode& t_node) {
    OAModule_Robot = std::make_shared<Robot> (obstacle);
    OAModule_ContMovement = std::make_shared<ContMovement> (OAModule_Robot, obstacle);
    std::shared_ptr<StmMovement> ContMovement_StmMovement = std::make_shared<StmMovement>(OAModule_Robot, OAModule_ContMovement, obstacle);
    OAModule_ContMovement->stm = ContMovement_StmMovement;

    // Epuck Sensors
    OAModule_Robot->EventsI::light_sensor_epuck = GetSensor<argos::CCI_EPuckLightSensor>("epuck_light");
    OAModule_Robot->EventsI::proximity_sensor_epuck = GetSensor<argos::CCI_EPuckProximitySensor>("epuck_proximity");
    // Epuck Actuators
    OAModule_Robot->MovementI::wheels_actuator = GetActuator<argos::CCI_EPuckWheelsActuator>("epuck_wheels");
    OAModule_Robot->MovementI::base_leds_actuator = GetActuator<argos::CCI_EPuckBaseLEDsActuator>("epuck_base_leds");
}

void OAModule::Execute() {
    OAModule_Robot->Sensors();
    OAModule_ContMovement->Execute();
    OAModule_Robot->Actuators();
}

void OAModule::ControlStep() {
    Execute();
    printf("\n");
}
```



