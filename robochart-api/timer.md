## Timer {#event}

In simulation, the smallest update unit for a timer is the control step. The simulation is executed in a cyclic way. All the timers are updated in the same phase, which means the clock is global.

```cpp
//Timer.h

#ifndef ROBOCALC_TIMER_H_
#define ROBOCALC_TIMER_H_

#include <string>

#define TIMER_DEBUG

namespace robochart {

class Timer {
public:

    Timer(std::string name) : name(name), counter(0), waitFlag(false), waitPeoriod(0), startingCounter(65535)
    {}

    int GetCounter() const {
        return counter;
    }

    void SetCounter(int i) {counter = i;}

    void IncCounter() {
        counter++;
        if (counter - startingCounter >= waitPeoriod) {
            waitFlag = false;
            startingCounter = 65535;
        }
        #ifdef TIMER_DEBUG
                printf("%s counter: %d\n", name.c_str(), counter);
        #endif
    }

    void Wait(int i) {
        waitFlag = true;
        waitPeoriod = i;
        startingCounter = counter;
    }

    bool CheckWaitStatus() {
        return waitFlag;
    }

    std::string GetName() {
        return name;
    }

private:
    int counter, waitPeoriod, startingCounter;
    bool waitFlag;
    std::string name;
};

}

#endif
```



