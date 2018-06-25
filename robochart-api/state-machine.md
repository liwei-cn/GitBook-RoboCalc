# State {#stateh}

The state machine components are defined in the`State.h`header file. It declares three classes: State, State Machine and Transition. A subclass of`State`can implement the following methods:

* `Entry()`: implement the entry action of the state;
* `During()`: implement the during action of the state;
* `Exit()`: implement the exit action of the state;
* `Initial()`: return the index of the initial substate.

The transitions of the state are stored in the variable`transitions`. The code for `State` and `Transition` class is as follows:

```cpp
//State.h

#ifndef ROBOCALC_STATE_H_
#define ROBOCALC_STATE_H_

#include <vector>
#include <memory>

#define STATE_DEBUG

namespace robochart {
class Transition;
class State {

public:
    std::string name;
    bool mark;
    State(std::string n) : name(n), stage(s_Inactive), mark(false) {}
    virtual ~State() { printf("Deleting state %s\n", name.c_str()); }
    virtual void Entry() {}
    virtual void During() {}
    virtual void Exit() {}
    virtual int Initial() { return -1; }

    enum Stages {
        s_Enter, s_Execute, s_Exit, s_Inactive
    };
    Stages stage;
    std::vector<std::shared_ptr<State>> states;
    std::vector<std::shared_ptr<Transition>> transitions;
    virtual void Execute();

    bool TryTransitions();

    bool TryExecuteSubstates(std::vector<std::shared_ptr<State>> s);

    void CancelTransitions(int i);
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
    std::string name;
public:
    Transition(std::string n, std::weak_ptr<State> src, std::weak_ptr<State> tgt) :
            name(n), source(src), target(tgt) {
    }
    virtual void Reg() {}
    virtual bool Check() { return true; }
    virtual void Cancel() {}
    virtual bool Condition() { return true; }
    virtual void Action() {}
    virtual void ClearEvent() {};
    virtual ~Transition() { source.reset(); target.reset(); printf("Deleting transition\n");}
    bool Execute();
};

}

#endif
```

If the state is composite, its substates are stored in the variable`states`. In this case, the function`Initial()`must be implemented and return the index of the initial state. Any state \(e.g. a state machine\) must extend the class State and can provide entry, during and exit actions.

Notice that it is not possible to define the transitions of the state directly in the class because of a circular dependency between states and transitions. For this reason, transitions must be instantiated and added to the variable`transitions`of the source state.

For example, in our obstacle avoidance example, the`Turning`state has reference to the robot \(to access the`Move`operation\), and the state machine \(to access variable `dir`  and clock `T`\); it provides the entry action that sets the angular speed of the robot.

## State Machine {#state-machine}

A state machine is just a state. This class exists only to make the notion of a state machine explicit. To execute a state machine, the `execute` function is called. Inside this function, the functions try\__execute\_substates, try_\_transitions,  cancel\_transitions are called. A state has three stages: `s_Enter`, `s_Execute` and `s_Exit`.

```cpp
//State.cpp
#include "State.h"

namespace robochart {

bool Transition::Execute() {
    if (Condition() & Check()) {  //check condition() first if it is false no need to perform check(); condition() && check()
        auto src = source.lock();
        src->stage = State::s_Exit;
        src->Execute();
        Action();
        auto tgt = target.lock();
        tgt->stage = State::s_Enter;
        tgt->Execute();
        return true;
    }
    return false;
}

void State::Execute() {
    switch (stage) {
    case s_Enter:
#ifdef STATE_DEBUG
        printf("Entering State %s\n", this->name.c_str());
#endif
        Entry();
        if (Initial() >= 0) {
            states[Initial()]->stage = s_Enter;  //this has already makes sure that every time the state machine is entered, it starts executing from initial state?
            states[Initial()]->Execute();
        }
        stage = s_Execute;
        break;
    case s_Execute:
#ifdef STATE_DEBUG
        printf("Executing a state %s\n", this->name.c_str());
#endif
        while(TryExecuteSubstates(states));      //this makes sure more than one transition can happen at one cycle; execute the state from bottom to up
        if (TryTransitions() == false) {
#ifdef STATE_DEBUG
            printf("Executing during action of %s!\n", this->name.c_str());
#endif
            During();                              //if no transition is enabled, execute during action in every time step
        }
        else {
#ifdef STATE_DEBUG
            printf("Not Executing during action of %s!\n", this->name.c_str());
#endif
        }
        break;
    case s_Exit:
        Exit();
        stage = s_Inactive;
        break;
    }
}

bool State::TryTransitions() {
#ifdef STATE_DEBUG
    printf("trying %ld transitions\n", transitions.size());
#endif
    for (int i = 0; i < transitions.size(); i++) {
#ifdef STATE_DEBUG
        printf("trying transition: %s\n", transitions[i]->name.c_str());
#endif
        bool b = transitions[i]->Execute();
        if (b) {
            this->mark = true;
            CancelTransitions(i);  //erase OTHER events (in the channel) already registered by the transitions of this state, as the state tried its every possible transitions
#ifdef STATE_DEBUG
            printf("transition %s true\n", transitions[i]->name.c_str());
#endif
            return true;
        }
        else {
#ifdef STATE_DEBUG
            printf("transition %s false\n", transitions[i]->name.c_str());
#endif
        }
    }
    this->mark = false;
    return false;
}

void State::CancelTransitions(int i) {
    for (int j = 0; j < transitions.size(); j++) {
        if (j != i) {
#ifdef STATE_DEBUG
            printf("CANCEL transition: %s\n",transitions[j]->name.c_str());
#endif
            transitions[j]->Cancel();
        }
    }
}

//return false either no sub states or no transitions are enabled in the sub states
bool State::TryExecuteSubstates(std::vector<std::shared_ptr<State>> s) {
    for (int i = 0; i < s.size(); i++) {
        // printf("state index : %d; stage: %d\n", i, states[i]->stage);
        // there should be only one active state in a single state machine
        if (s[i]->stage == s_Inactive) continue;
        else {
            s[i]->Execute();
            return s[i]->mark;      //keep trying the transitions at the same level if there is transition from one state to another
        }
    }
    return false;
}

}
```

## Transition {#transition}

Similarly to states, transitions must extend the class Transition. They can provide a number of functions:`reg`,`check`,`cancel`,`condition`and`action`. The first three are necessary if the transition has an event as a trigger, the fourth if the transition has a condition and the final if the transition has an action. A subclass of`Transition`may implement four optional functions:

* `virtual void Reg()`: this function is only implemented if the transition has a trigger with an event. In this case, the function`reg`of a channel must be called \(with appropriate parameters\) and the event returned by the call must be stored in a variable such as`event`;
* `bool Check()`: this function is implemented to check the occurrence of the registered event. It calls the`Check`function of the channel on the event produced by`Reg`;
* `void Cancel()`: this function call the method cancel of the channel with the event produced by`reg`as a parameter;
* `bool Condition()`: this function implemented the condition of the transition;
* `void Action()`: this function implements the action of the transition. If the transition's trigger contains an event, this function must call the function`Accept`or`AcceptAndDelete`of the channel \(on the event obtained from`Reg`\). The function`Accept`should be called if the channel is synchronous, and`AcceptAndDelete`should be called if the channel is asynchronous.

In the obstacle avoidance example, the transition `t1` has a trigger of the form`obstacle?dir`and also reset the clock `T`. This transition is implemented as the classt`1`, where`reg`is implemented as a function that calls the function`reg`of the channel`obstacle`of the state machine with source name `StmMovement` \(identifying the state machine StmMovement\) and undefined parameter`Optional<Loc>()`. The result of the`ref`function of the channel is a pointer to an event that is recorded in the variable`event`.

The`check`function simply returns the result of calling the`check`function of the channel`obstacle`. If the `check` function return true, the first parameter of the event to the state machine variable`dir`using the function`get<0>`applied to the event.  the event must be accepted to mark it as treated and set to the`nullptr`. In the case of the transition `t1`, the action finishes with a call to`acceptAndDelete`because the channel`obstacle`is asynchronous. If it were synchronous, the method`accept`should be called, which will only delete the event if both sides of the communication have accepted it. If the check function return false, the function`cancel`calls the function`cancel`of the same channel. All these calls are guarded by a check on the variable`event`that guarantees it is not null.

Transition `t2` illustrates the implementation of a transition with a condition.

The condition calculates the time steps elapsed recorded in `T` . The time step is increased in every cycle of the simulation followed by the execution of the state machine. If the time passes certain threshold, transition `t2` is trigger.

