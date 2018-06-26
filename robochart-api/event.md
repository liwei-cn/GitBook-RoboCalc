## Event {#event}

The `Event` class is used only as parameters to the channel function and to obtain the values associated with the event. The `Event` class has the attributes of a channel name, source name \(stating where it comes from\). A event can also have parameters in order to pass information \(via `send` e!x\), the value would be stored in the attribute of parameters. It is a tuple which means it can store multiple value rather than a single value. The value can be empty. In this case, when an event is registered, only the source name would be passed. The `Event` class also has an attribute of reference of the other paired event inside the channel. This value is only set when the events in the channel are matched.  The `Event` class also has a attribute accepted, marking whether an event is accepted/treated by the other component.

```cpp
//Event.h

#ifndef ROBOCALC_CHANNEL_H_
#define ROBOCALC_CHANNEL_H_

#include <stdlib.h>
#include <memory>
#include <mutex>
#include <set>
#include <iostream>
#include <algorithm>
#include "Event.h"

namespace robochart {

template<typename ...Args>
class Channel {
private:
    std::string name;
//    std::mutex m;
    std::vector<std::shared_ptr<Event<Args...>>>events;  //store the number of shared event pointer in the channel
public:
    Channel(std::string n):
    name(n) {
    }
    virtual ~Channel() {
    }
    /* return the size of the channel
     *
     */
    uint Size() {
        return events.size();
    }
    /* clear the channel
     *
     */
    void Clear() {
        events.clear();
    }

    /* Args ... args: the number of arbitrary type of arguments of the event;
     * In principle, there can be multiple numbers, but here we use one, which is Optional.
     * The Optional class includes the value of the argument
     */
    std::shared_ptr<Event<Args...>> Reg(std::string source, Args ... args) {
        //std::lock_guard<std::mutex> lck(m);
        std::shared_ptr<Event<Args...>> e = std::make_shared<Event<Args...>>(name,
                source, args...);    //the event in the channel is instantiated here: call Event constructor
        events.push_back(e);         //push the shared event pointer into the channel
#ifdef EVENT_DEBUG
        printf("channel %s size (registered): %ld\n", GetName().c_str(), events.size());
#endif
        return e;
    }

    //This is the overloaded function
    std::shared_ptr<Event<Args...>> Reg(std::shared_ptr<Event<Args...>> ci) {
        //std::lock_guard<std::mutex> lck(m);
        events.push_back(ci);
        return ci;
    }

    bool Check(std::shared_ptr<Event<Args...>> e) {
        //std::lock_guard<std::mutex> lck(m);
        for (typename std::vector<std::shared_ptr<Event<Args...>>>::iterator it = events.begin();
                it != events.end(); ++it) {
            #ifdef EVENT_DEBUG
                printf("channel %s size: %ld\n", GetName().c_str(), events.size());
            #endif
            if (e->Compatible(*it)) {
                e->Match(*it);
                (*it)->SetOther(e);   //e->match(*it) and (*it)->setOther(e) will make sure the matched events will have a reference to each other
#ifdef EVENT_DEBUG
                printf("checking of channel %s is true\n", GetName().c_str());
#endif
                return true;
            }
        }
#ifdef EVENT_DEBUG
        printf("checking of channel %s is false\n", GetName().c_str());
#endif
        return false;
    }
    void Cancel(std::shared_ptr<Event<Args...>> e) {
        //std::lock_guard<std::mutex> lck(m);
        if (!e->GetOther().Exists()) {
            typename std::vector<std::shared_ptr<Event<Args...>>>::iterator position = std::find(events.begin(), events.end(), e);
            if (position != events.end())
               events.erase(position);
#ifdef EVENT_DEBUG
            printf("event %s (%d) removed from channel\n", GetName().c_str(), e != nullptr);
#endif
        } else {
#ifdef EVENT_DEBUG
            printf("error removing event from channel %s\n", GetName().c_str());
#endif
        }
    }

    void Accept(std::shared_ptr<Event<Args...>> e) {    //This will create a temp new shared pointer which will be out of scope when the function terminates
        //std::lock_guard<std::mutex> lck(m);
        if (e->getOther().exists()) {
            e->accept();            //The first component will only accept the event (but not delete the event in the channel), because e->getOther().value().lock()->isAccepted() is false;
                                    //the second component will accept the event as well; but it will also delete both events in the channel, because e->getOther().value().lock()->isAccepted() becomes true.
            if (e->getOther().value().lock()->isAccepted()) {

                // The other has already been accepted so I can remove and reset both
                std::weak_ptr<Event<Args...>> other = e->getOther().value();
                typename std::vector<std::shared_ptr<Event<Args...>>>::iterator p1 = std::find(events.begin(), events.end(), e);
                if (p1 != events.end())
                    events.erase(p1);
                typename std::vector<std::shared_ptr<Event<Args...>>>::iterator p2 = std::find(events.begin(), events.end(), other.lock());
                if (p2 != events.end())
                    events.erase(p2);  //erase the share_ptr using weak_ptr.lock; the weak_ptr will expire when it is out of the if scope; in this case, both the shared_point are deleted in the channel
#ifdef EVENT_DEBUG
                printf("done accepting and reseting event\n");
#endif
            }
        }
    }

    void AcceptAndDelete(std::shared_ptr<Event<Args...>> e) {
        //std::lock_guard<std::mutex> lck(m);
        if (e->GetOther().Exists()) {      //delete both shared_ptr in the channel; if check() returns true; e->getOther().exists() will return true
            std::weak_ptr<Event<Args...>> other = e->GetOther().GetValue();
            typename std::vector<std::shared_ptr<Event<Args...>>>::iterator p1 = std::find(events.begin(), events.end(), e);
            if (p1 != events.end())
                events.erase(p1);
            typename std::vector<std::shared_ptr<Event<Args...>>>::iterator p2 = std::find(events.begin(), events.end(), other.lock());
            if (p2 != events.end())
                events.erase(p2);
        }
#ifdef EVENT_DEBUG
        printf("channel %s size after acceptance: %ld\n", GetName().c_str(), events.size());
#endif
    }

    std::string GetName() {
        return name;
    }
};

}

#endif
```

* `Compatible` Check whether two events in the channel come from different sources by comparing two strings. This means the communication is only valid between two different components. The state machine can not `send` an event that triggers a transition of its own. 
* `Match`Set the reference of each event. This is used for deleting the event from the channel after they are treated. If it is asynchronous communication, after the events are treated \(e.g. a transition is enabled\), both registered will be deleted from the channel by one side of the communication. In the obstacle avoidance example, the state machine will be responsible to clear the paired events from the channel.

## Optional {#event}

The `Optional`class is a template class. It can be used to define the associated value of an event. The event type can be any primitive or used-defined types. It is also used for define the other event as the paired event needs to have the reference to each other. Usage examples:

```cpp
Optional<double> doubleType
```

```cpp
Optional<std::weak_ptr<Event<Args...>>> otherEvent
```

```cpp
//optional.h

#ifndef ROBOCALC_OPTIONAL_H_
#define ROBOCALC_OPTIONAL_H_

namespace robochart {

template <typename T>
class Optional {
public:
    Optional(): set(false) {}
    Optional(T t): set(true),v(t) {}

    bool Exists() {
        return set;
    }
    T GetValue() {
        return v;
    }
    void SetValue(T t) {
        set = true;
        v = t;
        printf("### CHANGING VALUE OF OPTIONAL\n");
    }
    ~Optional() {}
private:
    T v;
    bool set;
};

}

#endif
```

We use the template class, as the type of the associated value the an event is unknown. Just like we can create** function templates**, we can also create **class templates**, allowing classes to have members that use template parameters as types.

