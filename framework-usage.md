# Framework Usage {#framework-usage}

![](/assets/OAModule.png)

A package called OAModule is created. This package defines a module `OAModule,`and the module includes a robotic platform `Robot`and a controller `ContMovement`. The robot provides one interface `MovementI` and uses one interface `EventsI.`The first interface groups the operation `Move` and a const variable `PI`, and the second interface  groups event channel `obstacle`. The robotic platform is composed of a controller `ContMovement` that contains the state machine `StmMovement`. The state machine declares a variable `dir` and the clock `T` . It consists of two states \(`Moving` and `Turning`\), and the first uses the operation to move the robot forward. If an obstacle is found, the clock is reset and the state `Turning` is entered. In the `Turning` state, the robot starts turning based on the location of the obstacle, and if the time passed a certain threshold PI, the robot enters the `Moving` state again.

## Module {#module}

The Module class instantiates the robot, the controller and all the channels used in the model.

```cpp
//OAModule.h
class OAModule {
public:
    OAModule() :
            OAModule_Robot(nullptr),
            OAModule_ContMovement(nullptr),
            obstacle(std::make_shared<robochart::obstacle_channel>("obstacle")) {}
    virtual ~OAModule() {};

    void Init();
    void Execute();

private:
    std::shared_ptr<robochart::obstacle_channel> obstacle;
    std::shared_ptr<Robot> OAModule_Robot;
    std::shared_ptr<ContMovement> OAModule_ContMovement;
};
```

It also implements the functions`Init`and `Execute`. Init function instantiates the `OAModule_Robot` and `OAModule_ContMovement`class. This function can also be extended to pass initial parameters to the robot and controller. In ARGoS, this is done via reading a XML file using the Init function. The function `Execute` executes the sensors of the robot, the behaviour of the controller and then executes the actuators of the robot. This function can be executed in the step function of a particular simulator. In ARGoS, the step function is called `ControlStep()`.

## RobotHardware {#robot}

The `HardwareComponent`class is a abstract class that include two functions Sensors and Actuators. These two functions need to be extended.

```cpp
//HardwareComponent.h
class HardwareComponent {

public:
    HardwareComponent() {}
    virtual ~HardwareComponent() {}
    virtual void Sensors() = 0;
    virtual void Actuators() = 0;
};
```

## Interface {#robot}

Each of the interfaces is declared as a class that implements`HardwareComponent`.

```cpp
//EventsI.h
class EventsI : public HardwareComponent {
public:
    EventsI(std::shared_ptr<robochart::obstacle_channel> obstacle);
    virtual ~EventsI();

    virtual void Sensors();
    virtual void Actuators();


private:
    std::shared_ptr<robochart::obstacle_channel> obstacle;

};
```

```cpp
//MovementI.h
class MovementI : public HardwareComponent {
public:
    MovementI();
    virtual ~MovementI();
    void Move(double lv, double av);

    virtual void Sensors();
    virtual void Actuators();

public:
    int PI;


};
```

The`Move`function provided by the interface sets the desired linear and angular speed of the robot. In the Actuators function, the desired linear and angular speed is used to calculate the required velocity of the left and right wheels to activate the motor. The class`MovementI`or `EventsI` requires the implementation of the functions`Sensors`and`Actuators`. This is the wrapper that maps the notions of RoboChart to the native simulation.

## Robot {#robot}

The robot inherits two interfaces:`MovementI`and `EventsI` that provide the operation`Move`and event in the channel`obstacle`, respectively.

```cpp
//Robot.h
class Robot: public MovementI, public EventsI {
public:
    Robot(
        std::shared_ptr<robochart::obstacle_channel> obstacle);
    virtual ~Robot();

    void Sensors();
    void Actuators();

private:
    std::shared_ptr<robochart::obstacle_channel> obstacle;

};
```

The constructor instantiates each of the interfaces, and the functions`Sensors`and`Actuators`execute the respective functions of each of the interfaces.

## Controller {#controller}

A controller class`ContMovement` implements a generic Controller class that contains a reference to a state machine.

```cpp
//ContMovement.h
class ContMovement: public robochart::Controller {
public:
    ContMovement(
            std::shared_ptr<Robot> R, 
            std::shared_ptr<robochart::obstacle_channel> obstacle);
    virtual ~ContMovement();
    virtual void Execute();

private:
    std::shared_ptr<robochart::obstacle_channel> obstacle;
private:
    std::shared_ptr<Robot> R;

};
```

The `ContMovement` controller receives pointers to the robot, and one channel`obstacle`. The`stm`field needs to be set by the module in the Init function to avoid issues with circular dependencies.

## State Machine {#state-machine}

A state machine is a class that inherits from`StateMachine`, instantiates its substates and transitions and links them together. The classes that implement the states and transitions of the machine are presented in the next section.

```cpp
//StmMovement.h
class StmMovement: public robochart::StateMachine {
private:
    std::shared_ptr<robochart::obstacle_channel> obstacle;
private:
    std::shared_ptr<robochart::Timer> T;
    std::shared_ptr<Robot> R;
    std::shared_ptr<ContMovement> C;

public:
     Loc dir;
public:
    StmMovement(std::shared_ptr<Robot> R, std::shared_ptr<ContMovement> C, std::shared_ptr<robochart::obstacle_channel> obstacle);
    ~StmMovement();
    int initial();
    virtual void execute();

public:
    class Moving : public robochart::State {
    public:
        Moving(std::shared_ptr<Robot> R, std::shared_ptr<ContMovement> C, std::shared_ptr<StmMovement> S) : State("Moving"), R(R), C(C), S(S) {}

    void entry() {
        R->Move(5, 0);
    }
    private:
        std::shared_ptr<Robot> R;
        std::shared_ptr<ContMovement> C;
        std::shared_ptr<StmMovement> S;
    };
    class Turning : public robochart::State {
    public:
        Turning(std::shared_ptr<Robot> R, std::shared_ptr<ContMovement> C, std::shared_ptr<StmMovement> S) : State("Turning"), R(R), C(C), S(S) {}

    void entry() {
        if (S->dir == Loc::left) {
            R->Move(0, 5);
        }
        else {
            R->Move(0, -5);
        }
    }
    private:
        std::shared_ptr<Robot> R;
        std::shared_ptr<ContMovement> C;
        std::shared_ptr<StmMovement> S;
    };

public:
    class t1 : public robochart::Transition {
        private:
            std::shared_ptr<Robot> R;
            std::shared_ptr<ContMovement> C;
            std::shared_ptr<StmMovement> S;
            std::shared_ptr<robochart::obstacle_event> event;
        public:
            t1(std::shared_ptr<Robot> R, std::shared_ptr<ContMovement> C, std::shared_ptr<StmMovement> S, std::weak_ptr<robochart::State> src, std::weak_ptr<robochart::State> tgt):
               robochart::Transition(src, tgt), R(R), C(C), S(S), event(nullptr)
            {}
            void reg() {
                if (event == nullptr) {
                    event = S->obstacle->reg("StmMovement", robochart::optional<Loc>());
                }
            }
            bool check() {
                if (S->obstacle->check(event) == true) {
                    S->dir = std::get<0>(*event->getParameters()).value();
                    ClearEvent();
                    S->T->SetCounter(0);
                    return true;
                }
                else {
                    cancel();
                    return false;
                }
            }
            void cancel() {
                if (event != nullptr) {
                    S->obstacle->cancel(event);
                    event = nullptr;
                }
            }
            void ClearEvent() {
                S->obstacle->acceptAndDelete(event);
                event = nullptr;
            }

    };
    class t2 : public robochart::Transition {
        private:
            std::shared_ptr<Robot> R;
            std::shared_ptr<ContMovement> C;
            std::shared_ptr<StmMovement> S;
        public:
            t2(std::shared_ptr<Robot> R, std::shared_ptr<ContMovement> C, std::shared_ptr<StmMovement> S, std::weak_ptr<robochart::State> src, std::weak_ptr<robochart::State> tgt):
               robochart::Transition(src, tgt), R(R), C(C), S(S)
            {}

            bool condition() {
                if (S->T->GetCounter() >= R->PI) {
                    return true;
                }
                else {
                    return false;
                }
            }
    };
};
```

A state machine is essentially treated as a composite state with a vector of substates. Notice that the instantiation of state machine is incomplete. It is declared without its substates and transitions in order to simplify the constructor and also due to a circular dependency between states and transitions. Therefore, in the constructor of the state machine, its substates should be instantiated.

States should be instantiated bottom up, and transitions should be instantiated after the source and target states are instantiated, and added to their respective source states.

###  {#state}

###  {#transition}



