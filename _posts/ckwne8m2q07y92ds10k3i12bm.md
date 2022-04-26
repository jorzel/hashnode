## Ports and adapters architecture. Python + Nameko microexample

## Introduction
Port and adapters (or hexagonal) architecture is a software design concept introduced by Alistair Cockburn in 2005. The main goal of it is to provide a clear seperation between application logic and external dependencies like database, user interface, framework providing HTTP requests, etc. 

![paa.drawio.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1638306972626/Ti6CC5PhF.png)

Application core, that is agonostic about external services and dependencies, should provide orchestration of whole business process exploiting existing ports. There are two types of ports:
- primary port, that is an entry exposing application core to outside world. It is usually a fascade **called** by a primary adapter (e.g. REST API, CLI, etc.),
- secondary port, enables application core to communicate with external world (e.g. database, mail sender, etc). It is an interface that is **implemented ** by a secondary adapter.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1638306569956/jwHimA9Nkw.png)

## Implementation
 [Nameko](https://nameko.readthedocs.io/en/stable/what_is_nameko.html) is a python framework for building microservices providing simple HTTP requests, RPC and Messaging over AMQP protocol.
In our simple example we implement an application that register entries to the system and emit event that should be handled in other part of the system (or other microservice).

We define a **secondary port** as an interface that enables publishing domain events from application core. Depending on implementation it could publish messages to local event bus or genuine event broker like RabbitMQ (but implementation is a resposibility of an adapter).
```python
from abc import ABC, abstractmethod

class EventPublisher(ABC):
    @abstractmethod
    def publish(self, events):
        pass
```

To implement **secondary adapter** for ```EventPublisher``` we make a wrapper for Nameko ```EventDispatcher``` class.
```python
# domain/events.py
from datetime import datetime
from typing import Dict
from dataclasses import dataclass

@dataclass(frozen=True)
class DomainEvent:
    @property
    def as_dict(self) -> Dict:
        serialized = asdict(self)
        serialized["name"] = self.name
        return serialized

@dataclass(frozen=True)
class RegisteredEntryEvent(DomainEvent):
      email: str 
      name: str = 'RegisteredEntryEvent'
      timestamp: datetime = datetime.now()

# infrastructure/event_publisher.py
from typing import List
from nameko.events import EventDispatcher
from application.event_publisher import EventPublisher
from domain.event import DomainEvent

class NamekoEventPublisher(EventPublisher):
    def __init__(self, dispatcher: EventDispatcher):
        self._dispatcher = dispatcher

    def publish(self, events: List[DomainEvent]) -> None:
        for event in events:
            self._dispatcher(event.name, event.as_dict)
```

**Primary port** in our example is a simple application service defining registration use case. ```EventPublisher``` interface (not implementation) is injected into the service.
```python
# application/service.py
from application.event_publisher import EventPublisher
from domain.event import RegisteredEntryEvent

class RegistrationService:
    def __init__(self, event_publisher: EventPublisher):
        self._event_publisher = event_publisher

    def register_entry(self, email: str) -> None:
        event = RegisteredEntryEvent(email=email)
        # this should some business / domain stuff executed
        self._event_publisher.publish(event)
```

Nameko HTTP endpoint is a **primary adapter** for our ```RegistrationEntryService``` port. It handles HTTP request data and execute the service method.
```python
import json
from nameko.events import EventDispatcher
from nameko.web.handlers import http
from werkzeug.wrappers import Request, Response

class NamekoRegistrationService:
    name = "nameko_registration_service"
    dispatcher = EventDispatcher()

    @http("POST", "/register")
    def register_entry(self, request: Request) -> Response:
        request_params = json.loads(request.data)
        email = request_params["email"]
        service = RegistrationService(
            event_dispatcher=NamekoEventPublisher(self.dispatcher),
        )
        service.register_entry(email)
        return Response(f"Registered entry for {email=}")
```

The crucial thing here is that our application core does not have any knowledge about infrastructure and API, it operates only on interfaces. If we need a database access, we should define a repository interface and injected it into our service. Or, if there is a need for notification sending, we should make a notification sender interface that can be implemented by SMS or Email sender adapter in the infrastructure layer. But inside the application core we know nothing about infrastructure implementations (thanks to it the application logic can be easily unit tested). And this is the main gain of using this architecture.

## Summary
It was a microexample of hexagonal architecture base concepts using Python and Nameko. If you find it interesting, I recommend you to visit my github repository for extended implementation (including also a domain layer that was omitted here, for the sake of simplicity) of similar project (also Python, Nameko and Port and Adapters): https://github.com/jorzel/opentable. For more theoretical background about hexagonal architecture, go [here](https://herbertograca.com/2017/09/14/ports-adapters-architecture/).