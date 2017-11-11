# Controller {#controllerh}

A controller contains a single reference to a state machine that must be instantiated in the variable`stm`by the function`init`in the Module class. This is the only method that must be defined by the controller class. It is used to instantiate states and transitions, build the state machine and initialise the`stm`variable.

```cpp
//Controller.h
#include "State.h"

namespace robochart {

class Controller {
public:
    std::shared_ptr<StateMachine> stm;
    Controller();
    virtual ~Controller();
    virtual void Execute();
};

}
```



