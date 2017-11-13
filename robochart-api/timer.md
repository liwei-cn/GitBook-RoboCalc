## Timer {#event}

In simulation, the smallest update unit for a timer is the control step. The simulation is executed in a cyclic way. All the timers are updated in the same phase, which means the clock is global.

```cpp
//Timer.h

#ifndef ROBOCALC_TIMER_H_
#define ROBOCALC_TIMER_H_

namespace robochart {
class Timer {
public:

	Timer(double t = 0.1) : counter(0), waitFlag(false), waitPeoriod(0), startingCounter(65535), step(0.1)
	{}

	int GetCounter() const {
		return counter;
	}

	double GetTime() const {
		return counter*step;
	}
	void SetCounter(int i) {counter = i;}

	void IncCounter() {
		counter++;
		if (counter - startingCounter >= waitPeoriod) {
			waitFlag = false;
			startingCounter = 65535;
		}
		printf("counter: %d\n", counter);
	}

	void Wait(int i) {
		waitFlag = true;
		waitPeoriod = i;
		startingCounter = counter;
	}

	bool CheckWaitStatus() {
		return waitFlag;
	}

private:
	int counter, waitPeoriod, startingCounter;
	bool waitFlag;
	double step;
};

}

#endif
```



