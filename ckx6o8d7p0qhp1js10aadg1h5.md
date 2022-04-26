## Coupling. Two perspectives

Coupling is a concept used in software engineering to define how tight is a relationship between system components (classes, modules, subsystems). Here you have some defintions:
- > Coupling is inter-connection between modules </br>
source: K. Hennney: https://www.youtube.com/watch?v=tMW08JkFrBA

- > Loose Coupling is when things which should not be together are not together </br>
source: https://injulkarnilesh.github.io/design-principles/COHESION_AND_COUPLING/

- > Coupling represents the degree to which a single unit is independent from others </br> source: https://enterprisecraftsmanship.com/posts/cohesion-coupling-difference/

- >  Coupling refers to how inextricably linked different aspects of an application are </br> source: https://deviq.com/principles/single-responsibility-principle

## Two perspectives
Coupling is strictly connected to cohesion concept ("togetherness" of a component) and there is a common heuristic for software developers that we should design components that have high cohesion and are loosely coupled.
But the question is how can we determine level of coupling for a given component? Above definitions don't bring the straightforward answer. To answer this question we can divide coupling concept into two aspects: **quantitative** and **qualitative**.

Quantitative perspective refers to the number of dependencies that component A has. It can be represented by two metrics:
- Afferent coupling (Ca) - the number of external components that depend on component A.
- Efferent coupling (Ce) -  the number of external components that component A is dependent on.

![coupling.drawio.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1639513511372/UXsnRxp7I.png)

The second perspective is a qualitative and defines how tight is a dependency between component A and component B. This aspect is often categorized as a type of coupling, like control couping, data coupling, stamp coupling, etc. However, all these types usually are a gradation of knowledge. The more knowledge component A must have about component B to perform action P, the coupling between them is tighter.

## Decoupling

As far as  quantitative coupling is concerned, we know what must be done to make coupling looser from that perspective. We should decrease number of dependencies.
But how decrease qualitative coupling?
To present a spectrum of coupling states: from tight to loose, we prepared a set of implementations of a use-case: booking table in a restaurant. There is a main responsibility that is a domain action of booking table in a system, and the supportive action of sending notification to a person that booked it. 
In the following examples we use also four questions to determine how implementation of sending notification is coupled with business action. The questions are:
1. What supportive action is done? Does my component know which method / function is called?
2. How the supportive action is done? Does my component know the implementation?
3. Where the supportive action executor is instatiated? Does my component instatiate executor? Is it a global object? Is it injected?
4. What is the supportive action executor type? Is it a function? Is it a class or an interface?

The more 'yes' answers for question: 'do you know it?', the coupling is tighter.

### Internal method
```python
class BookingTableService:
    def book_table(restaurant_id: str) -> None:
        # some implemenation
        self.send_notification(restaurant_id)

    def send_notification(self, restaurant_id: str) -> None:
        # some implementation

BookingTableSerice().book_table(1)
```
1. (Do we know) What supportive action is done? **Yes**,  we know we a send a notification, because we call `send_notification` method.
2. (Do we know) How the supportive action is done? **Yes**, because we have implementation of `send_notification` inside our service class.
3. (Do we know) Where the supportive action executor is instatiated? **Yes**, it is a part of the service class.
4. (Do we know)  What is the supportive action executor type? **Yes**, yes it is`BookingTableService` type.

Here we don't have external dependency, but we put into the same component two different resposibilites. We know everything about sending notification, so coupling is huge.

### Delegation
```python
class NotificationSender:
    def send(self, **kwargs)-> None:
        # some implementation

class BookingTableService:
    def __init__(self):
        self._notification_sender = NotificationSender()

    def book_table(restaurant_id: str) -> None:
        # some implemenation
        self._notification_sender.send(recruitment_id=recruitment_id)

BookingTableSerice().book_table(1)
```
1. (Do we know) What supportive action is done? **Yes**,  we know we send a notification, because we call `NotificationSender.send_notification` method.
2. (Do we know) How the supportive action is done? **No**, because implementation is in the `NotificationSender` class.
3. (Do we know) Where the supportive action executor is instatiated? **Yes**, we instatiate in the constructor of the service.
4. (Do we know) What is the supportive action executor type? **Yes**, yes it is the `NotificationSender` type.

This is example of a hidden dependency. We instatiate it inside the service, and it is not represented in the service interface (so not "visible" from outside the component).

### Delegation with Dependency Injection
```python
class NotificationSender:
    def send(self, **kwargs) -> None:
        # some implementation

class BookingTableService:
    def __init__(self, notification_sender: NotificationSender):
        self._notification_sender = notification_sender

    def book_table(restaurant_id: str) -> None:
        # some implemenation
        self._notification_sender.send(recruitment_id=recruitment_id)

BookingTableSerice(notification_sender=NotificationSender()).book_table(1)
```
1. (Do we know) What supportive action is done? **Yes**,  we know we send a notification, because we call `NotificationSender.send_notification` method.
2. (Do we know) How the supportive action is done? **No**, because implementation is in `NotificationSender` class.
3. (Do we know) Where the supportive action executor is instatiated? **No**, instance of `NotificationSender` is injected into the service.
4. (Do we know) What is the supportive action executor type? **Yes**, yes it is the `NotificationSender` type.

Now we know less about instantiation of a sender and how it is implemented, because it is injected into our service. However, we still know what action is done and what is the concrete implementation of the action executor (what, not how).

### Delegation with Dependency Injection of Interface
```python
from abc import ABC, abstractmethod

class NotificationSender(ABC):
    @abstractmethod
    def send(self, **kwargs) -> None:
        pass

class SmsSender(NotificationSender):
    def send(self, **kwargs) -> None:
        # some implementation

class BookingTableService:
    def __init__(self, notification_sender: NotificationSender):
        self._notification_sender = notification_sender

    def book_table(restaurant_id: str) -> None:
        # some implemenation
        self._notification_sender.send(recruitment_id=recruitment_id)

BookingTableSerice(notification_sender=SmsSender()).book_table(1)
```
1. (Do we know) What supportive action is done? **Yes**,  we know we send a notification, because we call `NotificationSender.send_notification` method.
2. (Do we know) How the supportive action is done? **No**, because implementation is in `NotificationSender` class.
3. (Do we know) Where the supportive action executor is instatiated? **No**, instance of `NotificationSender` is injected into the service.
4. (Do we know) What is the supportive action executor type? **No**, we have only an interface, not concrete type.

In this approach, we have interface injected instead of concrete type. We have defined contract (interface) for notification sender and we can take any class that fulfill it.

### Event Dispatching
```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import datetime
from typing import List

@dataclass(frozen=True)
class Event:
    name: str
    originatior_id: str
    timestamp: datetime = datetime.now()


class EventHandler:
    def handle(self, events: List[Events]) -> None:
        for event in events:
            if event.name == 'booked_table':
                # some implementation of sending notification

class EventDispatcher(ABC):
    @abstractmethod
    def publish(self, events: List[Event]) -> None:
        pass

class LocalEventDispatcher(EventDispatcher):
    def publish(self, events: List[Event]) -> None:
        handler = EventHandler()
        handler.handle(events)

class BookingTableService:
    def __init__(self, event_dispatcher: EventDispatcher):
        self._event_dispatcher = event_dispatcher

    def book_table(restaurant_id: str) -> None:
        # some implemenation
        events = [Event(name="booked_table", originator_id=restaurant_id)]
        self._event_dispatcher.publish(events)

BookingTableSerice(event_dispatcher=LocalEventDispatcher()).book_table(1)
```
1. (Do we know) What supportive action is done? **No**,  we only dispatch event, do not delegate to send a notification explicitly.
2. (Do we know) How the supportive action is done? **No**, because implementation is in `NotificationSender` class.
3. (Do we know) Where the supportive action executor is instatiated? **No**, instance of `NotificationSender` is injected.
4. (Do we know) What is the supportive action executor type? **No**, we have only an interface, not concrete type.

Here we have no dependency. We published an event and not even know whether other component handles it.

|                              | (Do we know) How the supportive action is done? | (Do we know) Where the supportive action executor is instatiated? | (Do we know) What is the supportive action executor type? | (Do we know) What supportive action is done? |
|------------------------------|-------------------------------------------------|-------------------------------------------------------------------|-----------------------------------------------------------|----------------------------------------------|
| Internal method              |                       Yes                       |                                Yes                                |                            Yes                            |                      Yes                     |
| Delegation                   |                        No                       |                                Yes                                |                            Yes                            |                      Yes                     |
| Delegation with DI           |                        No                       |                                 No                                |                            Yes                            |                      Yes                     |
| Delegation with DI interface |                        No                       |                                 No                                |                             No                            |                      Yes                     |
| Event dispatching            |                        No                       |                                 No                                |                             No                            |                      No                      |

## Summary
We have showed some ways of analysing coupling between two components (resposibilities). However we should be concious it is not always true that the less coupling, the better. In some cases we don't need an abstraction (interface), because we have only one implementation. Introducing Dependency Injection with Interface in that case would be overkill. Event dispatching is a great way to decouple several components, but it also introduce some complexity. When we see only publishing event in the part of code (but not see handling it), we lose some sense of causality in our code (what happen next). So you must always choose optimal solution for your case. Thanks for reading.
