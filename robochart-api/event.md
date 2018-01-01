## Event {#event}

The `Event` class is used only as parameters to the channel function and to obtain the values associated with the event. The `Event` class has the attributes of a channel name, source name \(stating where it comes from\). A event can also have parameters in order to pass information \(via `send` e!x\), the value would be stored in the attribute of parameters. It is a tuple which means it can store multiple value rather than a single value. The value can be empty. In this case, when an event is registered, only the source name would be passed. The `Event` class also has an attribute of reference of the other paired event inside the channel. This value is only set when the events in the channel are matched.  The `Event` class also has a attribute accepted, marking whether an event is accepted/treated by the other component.

```cpp
//Event.h

#include <vector>
#include <sstream>
#include <memory>

#include "optional.h"

#ifndef CHANNELITEM_H_
#define CHANNELITEM_H_

namespace robochart {

template<typename ...Args>
class Event {
private:
    std::string channel;
    std::string source;
    std::shared_ptr<std::tuple<Args...>> parameters;
    optional<std::weak_ptr<Event<Args...>>> other;
    bool accepted;
public:
    Event(std::string c, std::string id, Args ... args) :
            channel(c), source(id), other(), accepted(false), parameters(
                    std::make_shared<std::tuple<Args...>>(std::make_tuple(args...))) {
    }

    virtual ~Event() {
    }

    std::shared_ptr<std::tuple<Args...>> getParameters() {
        return parameters;
    }
    optional<std::string> getSource() {
        return source;
    }
    optional<std::weak_ptr<Event<Args...>>> getOther() {
        return other;
    }
    void setOther(std::weak_ptr<Event<Args...>> s) {
        other = optional<std::weak_ptr<Event<Args...>>>(s);
    }
    void accept() {
        accepted = true;
    }
    bool isAccepted() {
        return accepted;
    }

    bool compatible(std::shared_ptr<Event<Args...>> e) {
        optional<std::string> e1, e2;
        e1 = getSource();
        e2 = e->getSource();
        if (e1.exists() && e2.exists() && e1.value().compare(e2.value()) != 0) 
        {
            printf("### EVENTS ARE COMPATIBLE \n");
            return true;
        } else {
            return false;
        }
    }

    void match(std::shared_ptr<Event<Args...>> e) {
        if (!compatible(e))
            return;
        printf("### EVENT MATCH EVENTS\n");
        setOther(e);
    }

    std::string get_channel() {
        return channel;
    }
};
}

#endif /* CHANNELITEM_H_ */
```

* `compatible` Check whether two events in the channel come from different sources by comparing two strings. This means the communication is only validate between two different components. The state machine can not `send` an event that triggers a transition of its own. 
* `match`Set the reference of each event. This is used for deleting the event from the channel after they are treated. If it is asynchronous communication, after the events are treated \(e.g. a transition is enabled\), both registered will be deleted from the channel by one side of the communication. In the obstacle avoidance example,, the state machine will be responsible to clear the paired events from the channel.

## Optional {#event}

The `optional`class is a template class. It can be used to define the associated value of an event. The event type can be any primitive or used-defined types. It is also used for define the other event as the paired event needs to have the reference to each other. Usage examples:

```cpp
optional<double> doubleType
```

```cpp
optional<std::weak_ptr<Event<Args...>>> otherEvent
```

```cpp
//optional.h

#ifndef ROBOCALC_OPTIONAL_H_
#define ROBOCALC_OPTIONAL_H_

namespace robochart {

template <typename T>  //template class
class optional {
public:
    optional(): set(false) {}
    optional(T t): set(true),v(t) {}

    bool exists() {
        return set;
    }
    T value() {
        return v;
    }
    void setValue(T t) {
        set = true;
        v = t;
        printf("### CHANGING VALUE OF OPTIONAL\n");
    }
    ~optional() {}
private:
    T v;  //the value of this optional class; it can be the value associated with an event or otherEvent
    bool set;
};

}

#endif /* ROBOCALC_OPTIONAL_H_ */
```

We use the template class, as the type of the associated value the an event is unknown. Just like we can create** function templates**, we can also create **class templates**, allowing classes to have members that use template parameters as types.

