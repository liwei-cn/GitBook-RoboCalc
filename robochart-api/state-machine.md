# State {#stateh}

The state machine components are defined in the`State.h`header file. It declares three classes: State, State Machine and Transition. A subclass of`State`can implement the following methods:

* `entry`: implements the entry action of the state;
* `exit`: implements the exit action of the state;
* `initial`: returns the index of the initial substate.

The transitions of the state are stored in the variable`transitions`. The code for `State` and `Transition` class is as follows:

```cpp
//State.h
#include <vector>
#include <memory>
#include "Stages.h"

namespace robochart {
class Transition;
class State {

public:
    std::string name;
    State(std::string n) : name(n), stage(s_Inactive) {}
    virtual ~State() { printf("Deleting state %s\n", name.c_str()); }
    virtual void entry() {}
    virtual void during() {}
    virtual void exit() {}
    virtual int initial() { return -1; }

    Stages stage;
    std::vector<std::shared_ptr<State>> states;
    std::vector<std::shared_ptr<Transition>> transitions;
    virtual void execute();         

    bool try_transitions();

    bool try_execute_substates(std::vector<std::shared_ptr<State>> s);

    void cancel_transitions(int i);
};

class StateMachine: public State {
public:
    StateMachine(std::string n): State(n) {}
    virtual ~StateMachine() {}
};

class Transition {
private:
    std::weak_ptr<State> source, target;
public:
    Transition(std::weak_ptr<State> src, std::weak_ptr<State> tgt) :
            source(src), target(tgt) {

    }
    virtual void reg() {}
    virtual bool check() { return true; }
    virtual void cancel() {}
    virtual bool condition() { return true; }
    virtual void action() {}
    virtual void ClearEvent() {};
    virtual ~Transition() { source.reset(); target.reset(); printf("Deleting transition\n");}
    bool execute();
};

}
```

If the state is composite, its substates are stored in the variable`states`. In this case, the function`initial`must be implemented and return the index of the initial state. Any state \(e.g. a state machine\) must extend the class State and can provide entry and exit actions.

Notice that is it not possible to define the transitions of the state directly in the class because of a circular dependency between states and transitions. For this reason, transitions must be instantiated and added to the variable`transitions`of the source state.

For example, in our obstacle avoidance example, the`Turning`state has references to the robot \(to access the`Move`operation\), and the state machine \(to access variable `dir`  and clock `T`\); it provides the entry action that sets the angular speed of the robot.

## State Machine {#state-machine}

A state machine is just a state. This class exists only to make the notion of a state machine explicit. To execute a state machine, the `execute` function is called. Inside this function, the functions try\__execute\_substates, try_\_transitions,  cancel\_transitions are called. A state has three stages: `s_Enter`, `s_Execute` and `s_Exit`.

```cpp
//State.cpp
#include "State.h"

namespace robochart {

bool Transition::execute() {
    reg();
    if (condition() && check()) {
        auto src = source.lock();
        src->stage = s_Exit;
        src->execute();
        action();
        auto tgt = target.lock();
        tgt->stage = s_Enter;
        tgt->execute();
        return true;
    }
    return false;
}

void State::execute() {
    switch (stage) {
    case s_Enter:
        printf("Entering State %s\n", this->name.c_str());
        entry();
        if (initial() >= 0) {
            states[initial()]->stage = s_Enter;
            states[initial()]->execute();
        }
        stage = s_Execute;
        break;
    case s_Execute:
        printf("Executing a state %s\n", this->name.c_str());
        if (try_transitions() == false) {
            while(try_execute_substates(states));
        }
        break;
    case s_Exit:
        exit();
        stage = s_Inactive;
        break;
    }
}

bool State::try_transitions() {
    printf("trying %ld transitions\n", transitions.size());
    for (int i = 0; i < transitions.size(); i++) {
        bool b = transitions[i]->execute();
        if (b) {
            cancel_transitions(i);
            return true;
        }
    }
    return false;
}

void State::cancel_transitions(int i) {
    for (int j = 0; j < transitions.size(); j++) {
        if (j != i) {
            printf("CANCEL transition index: %d\n",j);
            transitions[j]->cancel();
        }
    }
}

bool State::try_execute_substates(std::vector<std::shared_ptr<State>> s) {
    for (int i = 0; i < s.size(); i++) {
        if (s[i]->stage == s_Inactive) continue;
        else {
            s[i]->execute();
            if (s[i]->stage == s_Inactive) { return true; }
            else { return false; }
        }
    }
    return false;
}

}
```

## Transition {#transition}

Similarly to a states, transitions must extend the class Transition. They can provide a number of functions:`reg`,`check`,`cancel`,`condition`and`action`. The first three are necessary if the transition has an event as a trigger, the fourth if the transition has a condition and the final if the transition has an action. A subclass of`Transition`may implement four optional functions:

* `virtual void reg()`: this function is only implemented if the transition has a trigger with an event. In this case, the function`reg`of a channel must be called \(with appropriate parameters\) and the event returned by the call must be stored in a variable such as`event`;
* `bool check()`: this function is implemented to check the occurrence of the registered event. It calls the`check`function of the channel on the event produced by`reg`;
* `void cancel()`: this function call the method cancel of the channel with the event produced by`reg`as a parameter;
* `bool condition()`: this function implemented the condition of the transition;
* `void action()`: this function implements the action of the transition. If the transition's trigger contains an event, this function must call the function`accept`or`acceptAndDelete`of the channel \(on the event obtained from`reg`\). The function`accept`should be called if the channel is synchronous, and`acceptAndDelete`should be called if the channel is asynchronous.

In the obstacle avoidance example, the transition `t1` has a trigger of the form`obstacle?dir`and also reset the clock `T`. This transition is implemented as the classt`1`, where`reg`is implemented as a function that calls the function`reg`of the channel`obstacle`of the state machine with source name `StmMovement` \(identifying the state machine StmMovement\) and undefined parameter`optional<Loc>()`. The result of the`ref`function of the channel is a pointer to an event that is recorded in the variable`event`.

The`check`function simply returns the result of calling the`check`function of the channel`obstacle`. If the `check` function return true, the first parameter of the event to the state machine variable`dir`using the function`get<0>`applied to the event.  the event must be accepted to mark it as treated and set to the`nullptr`. In the case of the transition `t1`, the action finishes with a call to`acceptAndDelete`because the channel`obstacle`is asynchronous. If it were synchronous, the method`accept`should be called, which will only delete the event if both sides of the communication have accepted it. If the check function return false, the function`cancel`calls the function`cancel`of the same channel. All these calls are guarded by a check on the variable`event`that guarantees it is not null.

Transition `t2` illustrates the implementation of a transition with a condition.

The condition calculates the time steps elapsed recorded in `T` . The time step is increased in every cycle of the simulation followed by the execution of the state machine. If the time passes certain threshold, transition `t2` is trigger.
