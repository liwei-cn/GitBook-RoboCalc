# Channel

A channel is instantiated using the class`Channel`instantiated with the types of the parameters. To simplify the usage of channels and events, we suggest declaring auxiliary types. For example, the obstacle channel and events are declared as, respectively, a Channel and an Event with an optional`string`parameters. The optional class allows the construction of expressions of the form`c?x`, where`x`is an input parameter. The channel is a buffer. It has a name and also a vector of reference of the events. The size of the buffer should be always kept to 2. This mean the communication is always one-to-one.

```cpp
//Channel.h

#ifndef ROBOCALC_CHANNEL_H_
#define ROBOCALC_CHANNEL_H_

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
    //return the size of the channel
    uint size() {
        return events.size();
    }
    //clear the channel
    void clear() {
        events.clear();
    }

    std::shared_ptr<Event<Args...>> reg(std::string source, Args ... args) {
        std::shared_ptr<Event<Args...>> e = std::make_shared<Event<Args...>>(name, source, args...);
        // printf("event value: %f\n", std::get<0>(*e->getParameters()).value());  //std::get<0>(*e->getParameters()): optional
        // printf("registering event uses %ld\n", e.use_count());  //1 ownership: e
        events.push_back(e);
        // printf("event stored in the set: %p", *events.begin()); //The channel stores the address of shared pointer; the address of the shared pointer pointing to the same object will have the same address
        // printf("event registered uses %ld\n", e.use_count());   //2 ownerships: e and the one stored in the channel; after 'return e', e goes out of scope, while another shared_pointer is stored in the channel
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
                (*it)->setOther(e); //e->match(*it) and (*it)->setOther(e) will make sure the matched event will have a reference to each other
                printf("checking is true\n");
                return true;
            }
        }
        printf("checking is false\n");
        return false;
    }
    void cancel(std::shared_ptr<Event<Args...>> e) {
        // printf("event is not null: %d\n", e != nullptr);
        // printf("condition is %d\n", !(e->getOther().exists()));
        if (!e->getOther().exists()) {
            // printf("removing event, occurrences %d\n", (int)events.count(e));
            typename std::vector<std::shared_ptr<Event<Args...>>>::iterator position = std::find(events.begin(), events.end(), e);
            if (position != events.end())  // == events.end() means the element was not found
               events.erase(position);
            printf("event removed from channel\n");
            //e.reset();                         //no need to reset the event, as it will go out of scope after the function call terminates; also if e is reset before erase, the erase wont be finished
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

#endif
```

* `reg`This function registers an event in the channel. That is, an event pointer is pushed into the channel buffer. Every time a transition is tried, the `reg`function will be called. 
* `check`This function checks whether two events in the channel are compatible or not. It return true if they are compatible; false vice versa. When the two events comes from two different sources and the type of the event is the same, these two events are considered as compatible. If `check`is successful, each event would have a reference to each other via the functions `match` and `setOther`.

* `cancel` This function remove and destroy the event from the channel buffer. For example, if an event is registered into the channel but `check` is false, then this event need to be removed from the channel.

* `accept` and `acceptAndDelete`These two functions remove the events from the channel if they are treated \(after `check`return true\). `accept` is used in the synchronous communication, and acceptAndDelete is used in the asynchronous communication.

* `clear`This will clear the channel and all the shared pointers in the channel will be destroyed. When you clear the vector, the shared pointers it contains are destroyed, and this action automatically de-allocates any encapsulated objects with no more shared pointers referring to them.

The entire purpose of smart pointers is that they manage memory for you, and the entire purpose of shared pointers is that the thing they point to is automatically freed when no more shared pointers point to it.

