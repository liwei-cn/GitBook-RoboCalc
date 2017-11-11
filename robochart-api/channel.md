# Channel

A channel is instantiated using the class`Channel`instantiated with the types of the parameters. To simplify the usage of channels and events, we suggest declaring auxiliary types. For example, the obstacle channel and events are declared as, respectively, a Channel and an Event with an optional`string`parameters. The optional class allows the construction of expressions of the form`c?x`, where`x`is an input parameter. The channel is a buffer. It has a name and also a vector of reference of the events. The size of the buffer should be always kept to 2. This mean the communication is always one-to-one. 

```cpp
//Channel.h

#ifndef RCCHANNEL_H_
#define RCCHANNEL_H_

#include <stdlib.h>
#include <memory>
#include <mutex>
#include <set>
#include "Event.h"
#include <iostream>
#include <algorithm>

namespace robochart {

template<typename ...Args>
class Channel {
private:
    std::string name;
    std::vector<std::shared_ptr<Event<Args...>>>events;
public:
    Channel(std::string n):
    name(n) {
    }

    virtual ~Channel() {
    }

    uint size() {
        return events.size();
    }

    void clear() {
        events.clear();
    }

    std::shared_ptr<Event<Args...>> reg(std::string source, Args ... args) {
        std::shared_ptr<Event<Args...>> e = std::make_shared<Event<Args...>>(name, source, args...);
        events.push_back(e);
        return e;
    }

    std::shared_ptr<Event<Args...>> reg(std::shared_ptr<Event<Args...>> ci) {
        events.push_back(ci);
        return ci;
    }

    bool check(std::shared_ptr<Event<Args...>> e) {
        for (typename std::vector<std::shared_ptr<Event<Args...>>>::iterator it = events.begin();
                it != events.end(); ++it) {
            if (e->compatible(*it)) {
                e->match(*it);
                (*it)->setOther(e);
                printf("checking is true\n");
                return true;
            }
        }
        printf("checking is false\n");
        return false;
    }
    void cancel(std::shared_ptr<Event<Args...>> e) {
        if (!e->getOther().exists()) {
            typename std::vector<std::shared_ptr<Event<Args...>>>::iterator position = std::find(events.begin(), events.end(), e);
            if (position != events.end())
               events.erase(position);
            printf("event removed from channel\n");
        } else {
            printf("error removing event\n");
        }
    }
    void accept(std::shared_ptr<Event<Args...>> e) {
        if (e->getOther().exists()) {
            e->accept();            
            if (e->getOther().value().lock()->isAccepted()) {
                std::weak_ptr<Event<Args...>> other = e->getOther().value();
                typename std::vector<std::shared_ptr<Event<Args...>>>::iterator p1 = std::find(events.begin(), events.end(), e);
                  if (p1 != events.end())
                      events.erase(p1);
                typename std::vector<std::shared_ptr<Event<Args...>>>::iterator p2 = std::find(events.begin(), events.end(), other.lock());
                      if (p2 != events.end())
                          events.erase(p2);
                printf("done accepting and reseting event\n");
            }
        }
    }
    void acceptAndDelete(std::shared_ptr<Event<Args...>> e) {
        if (e->getOther().exists()) {
            std::weak_ptr<Event<Args...>> other = e->getOther().value();
            typename std::vector<std::shared_ptr<Event<Args...>>>::iterator p1 = std::find(events.begin(), events.end(), e);
              if (p1 != events.end())
                  events.erase(p1);
            typename std::vector<std::shared_ptr<Event<Args...>>>::iterator p2 = std::find(events.begin(), events.end(), other.lock());
                  if (p2 != events.end())
                      events.erase(p2);
        }
    }
    std::string getName() {
        return name;
    }
};

}

#endif /* RCCHANNEL_H_ */
```

* `reg`This function registers an event in the channel. That is, an event pointer is pushed into the channel buffer. Every time a transition is tried, the `reg`function will be called. 
* `check`This function checks whether two events in the channel are compatible or not. It return true if they are compatible; false vice versa. When the two events comes from two different sources and the type of the event is the same, these two events are considered as compatible. If `check`is successful, each event would have a reference to each other via the functions `match` and `setOther`.

* `cancel` This function remove and destroy the event from the channel buffer. For example, if an event is registered into the channel but `check` is false, then this event need to be removed from the channel.

* `accept` and `acceptAndDelete`These two functions remove the events from the channel if they are treated \(after `check`return true\). `accept` is used in the synchronous communication, and acceptAndDelete is used in the asynchronous communication.

* `clear`This will clear the channel and all the shared pointers in the channel will be destroyed. When you clear the vector, the shared pointers it contains are destroyed, and this action automatically de-allocates any encapsulated objects with no more shared pointers referring to them.

The entire purpose of smart pointers is that they manage memory for you, and the entire purpose of shared pointers is that the thing they point to is automatically freed when no more shared pointers point to it.