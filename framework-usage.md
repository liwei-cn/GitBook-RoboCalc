# Framework Usage {#framework-usage}

![](/assets/OAModule.png)

A package called OAModule is created. This package defines a module `OAModule,`and the module includes a robotic platform `Robot`and a controller `ContMovement`. The robot provides one interface `MovementI` and uses one interface `EventsI.`The first interface groups the operation `Move` and a const variable `PI`, and the second interface  groups event  `obstacle`. The robotic platform is composed of a controller `ContMovement` that contains the state machine `StmMovement`. The state machine declares a variable `dir` and the clock `T` . It consists of two states \(`Moving` and `Turning`\), and the first uses the operation to move the robot forward. If an obstacle is found, the clock is reset and the state `Turning` is entered. In the `Turning` state, the robot starts turning based on the location of the obstacle, and if the time passed a certain threshold PI, the robot enters the `Moving` state again.

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

The `HardwareComponent`class is a abstract class that include two virtual functions `Sensors()` and `Actuators()`. These two functions need to be extended.

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

The`Move()`function provided by the interface sets the desired linear and angular speed of the robot. In the`Actuators()` function, the desired linear and angular speed is used to calculate the required velocity of the left and right wheels to activate the motor. The class`MovementI`or `EventsI` requires the implementation of the functions`Sensors()`and`Actuators()`. This is the wrapper that maps the notions of RoboChart to the native simulation.

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

The constructor instantiates each of the interfaces, and the functions`Sensors()`and`Actuators()`execute the respective functions of each of the interfaces.

## Controller {#controller}

A controller class`ContMovement` implements a generic Controller class that contains a reference to a state machine.

```cpp
//ContMovement.h
class ContMovement: public robochart::Controller {
public:
    ContMovement(
            std::shared_ptr<Robot> R_Robot, 
            std::shared_ptr<robochart::obstacle_channel> obstacle);
    virtual ~ContMovement();
    virtual void Execute();

private:
    std::shared_ptr<robochart::obstacle_channel> obstacle;
private:
    std::shared_ptr<Robot> R_Robot;

};
```

The `ContMovement` controller receives pointers to the robot, and one channel`obstacle`. The`stm`field needs to be set by the module in the `Init()` function of the module class to avoid issues with circular dependencies.

## State Machine {#state-machine}

A state machine is a class that inherits from an abstract class`StateMachine`, instantiates its substates and transitions and links them together. The classes that implement the states and transitions of the machine are presented in the next section.

```cpp
//StmMovement.h
#define SM_DEBUG

class StmMovement: public robochart::StateMachine
{
public:
	std::shared_ptr<robochart::obstacle_channel> obstacle;
public:
	std::shared_ptr<robochart::Timer> T;
	std::shared_ptr<Robot> R_Robot;
	std::shared_ptr<ContMovement> C_ContMovement;
public:
	Loc dir;
public:
	StmMovement(
			std::shared_ptr<Robot> R_Robot, 
			std::shared_ptr<ContMovement> C_ContMovement, 
			std::shared_ptr<robochart::obstacle_channel> obstacle);
	~StmMovement();
	int Initial();
	virtual void Execute();

public:
	class Moving : public robochart::State 
	{
		public:
			Moving(std::shared_ptr<Robot> R_Robot, std::shared_ptr<ContMovement> C_ContMovement, std::shared_ptr<StmMovement> S_StmMovement) : State("Moving"), R_Robot(R_Robot), C_ContMovement(C_ContMovement), S_StmMovement(S_StmMovement) 
			{
			}
			void Entry()
			{
				R_Robot->Move(5, 0);
			}
		private:
			std::shared_ptr<Robot> R_Robot;
			std::shared_ptr<ContMovement> C_ContMovement;
			std::shared_ptr<StmMovement> S_StmMovement;
	};
	class Turning : public robochart::State 
	{
		public:
			Turning(std::shared_ptr<Robot> R_Robot, std::shared_ptr<ContMovement> C_ContMovement, std::shared_ptr<StmMovement> S_StmMovement) : State("Turning"), R_Robot(R_Robot), C_ContMovement(C_ContMovement), S_StmMovement(S_StmMovement) 
			{
			}
			void Entry()
			{
				if (S_StmMovement->dir == Loc::left) 
				{
					R_Robot->Move(0, 5);
				}
				else 
				{
					R_Robot->Move(0, -5);
				}
			}
		private:
			std::shared_ptr<Robot> R_Robot;
			std::shared_ptr<ContMovement> C_ContMovement;
			std::shared_ptr<StmMovement> S_StmMovement;
	};
	class i0 : public robochart::State 
	{
		public:
			i0(std::shared_ptr<Robot> R_Robot, std::shared_ptr<ContMovement> C_ContMovement, std::shared_ptr<StmMovement> S_StmMovement) : State("i0"), R_Robot(R_Robot), C_ContMovement(C_ContMovement), S_StmMovement(S_StmMovement) 
			{
			}
		private:
			std::shared_ptr<Robot> R_Robot;
			std::shared_ptr<ContMovement> C_ContMovement;
			std::shared_ptr<StmMovement> S_StmMovement;
	};

	public:
		class t1 : public robochart::Transition {
			private:
				std::shared_ptr<Robot> R_Robot;
				std::shared_ptr<ContMovement> C_ContMovement;
				std::shared_ptr<StmMovement> S_StmMovement;
				std::shared_ptr<robochart::obstacle_event> reg_obstacle_event;
			public:
				t1(std::shared_ptr<Robot> R_Robot, std::shared_ptr<ContMovement> C_ContMovement, std::shared_ptr<StmMovement> S_StmMovement, std::weak_ptr<robochart::State> src, std::weak_ptr<robochart::State> tgt):
				   robochart::Transition("S_StmMovement_t1", src, tgt), R_Robot(R_Robot), C_ContMovement(C_ContMovement), S_StmMovement(S_StmMovement), reg_obstacle_event(nullptr)
				{}
				void Reg() {
					if (reg_obstacle_event == nullptr) {
						reg_obstacle_event = S_StmMovement->obstacle->Reg("StmMovement", robochart::Optional<Loc>());
					}
				}
				bool Check() {
					Reg();
					if (S_StmMovement->obstacle->Check(reg_obstacle_event) == true) {
						#ifdef SM_DEBUG
							printf("TREATING EVENT obstacle\n");
						#endif
						S_StmMovement->dir = std::get<0>(*reg_obstacle_event->GetOther().GetValue().lock()->GetParameters()).GetValue();
						ClearEvent();
						return true;
					}
					else {
						Cancel();
						return false;
					}
				}
				void Cancel() {
					if (reg_obstacle_event != nullptr) {
						S_StmMovement->obstacle->Cancel(reg_obstacle_event);
						reg_obstacle_event = nullptr;
					}
				}
				void ClearEvent() {
					S_StmMovement->obstacle->AcceptAndDelete(reg_obstacle_event);
					reg_obstacle_event = nullptr;
				}
				void Action() {
					S_StmMovement->T->SetCounter(0);
					#ifdef SM_DEBUG
						printf("Resetting Clock T\n");
					#endif
				}
		};
	public:
		class t2 : public robochart::Transition {
			private:
				std::shared_ptr<Robot> R_Robot;
				std::shared_ptr<ContMovement> C_ContMovement;
				std::shared_ptr<StmMovement> S_StmMovement;
			public:
				t2(std::shared_ptr<Robot> R_Robot, std::shared_ptr<ContMovement> C_ContMovement, std::shared_ptr<StmMovement> S_StmMovement, std::weak_ptr<robochart::State> src, std::weak_ptr<robochart::State> tgt):
				   robochart::Transition("S_StmMovement_t2", src, tgt), R_Robot(R_Robot), C_ContMovement(C_ContMovement), S_StmMovement(S_StmMovement)
				{}
				bool Condition() {
					if (S_StmMovement->T->GetCounter() >= R_Robot->PI) {
						#ifdef SM_DEBUG
							printf("Condition of transition S_StmMovement_t2 is true\n");
						#endif
						return true;
					}
					else {
						#ifdef SM_DEBUG
							printf("Condition of transition S_StmMovement_t2 is false\n");
						#endif
						return false;
					}
				}
		};
	public:
		class t0 : public robochart::Transition {
			private:
				std::shared_ptr<Robot> R_Robot;
				std::shared_ptr<ContMovement> C_ContMovement;
				std::shared_ptr<StmMovement> S_StmMovement;
			public:
				t0(std::shared_ptr<Robot> R_Robot, std::shared_ptr<ContMovement> C_ContMovement, std::shared_ptr<StmMovement> S_StmMovement, std::weak_ptr<robochart::State> src, std::weak_ptr<robochart::State> tgt):
				   robochart::Transition("S_StmMovement_t0", src, tgt), R_Robot(R_Robot), C_ContMovement(C_ContMovement), S_StmMovement(S_StmMovement)
				{}
		};
};
```

A state machine is essentially treated as a composite state with a vector of substates. Notice that the instantiation of state machine is incomplete. It is declared without its substates and transitions in order to simplify the constructor and also due to a circular dependency between states and transitions. Therefore, in the constructor of the state machine, its substates should be instantiated.

States should be instantiated bottom up, and transitions should be instantiated after the source and target states are instantiated, and added to their respective source states.

###  {#state}

###  {#transition}



