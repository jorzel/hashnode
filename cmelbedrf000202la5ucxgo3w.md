---
title: "Strategic DDD by Example: Bounded Contexts Mapping"
seoTitle: "Strategic DDD by Example: Bounded Contexts Mapping"
seoDescription: "Strategic DDD by Example: Bounded Contexts Mapping. Bounded contexts relationships patterns"
datePublished: Thu Aug 21 2025 11:24:59 GMT+0000 (Coordinated Universal Time)
cuid: cmelbedrf000202la5ucxgo3w
slug: strategic-ddd-by-example-bounded-contexts-mapping
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1755775319198/4434238e-b8d9-4ce8-b171-b0d8248f0b6b.png
tags: software-development, programming, software-architecture, software-engineering, domain-driven-design

---

This is the second part of our Strategic DDD series. I hoped to deliver it earlier, but it took me over 2 years to get it done. I hope it will be helpful and enjoyable.

In the first part, we explored how to identify subdomains within a business domain using a restaurant reservation system as our example. We discovered some real gains from domain decomposition: better modularity, team autonomy, improved scalability, and the ability to focus development efforts on the core subdomain. But now we need to face the music. There's an inevitable cost that comes with this separation.

## From Subdomains to Bounded Contexts

Before we jump into integration challenges, let's clear up something that trips up a lot of people. Subdomains and bounded contexts aren't the same thing. Understanding this difference matters if you want to avoid confusion later.

A subdomain is part of the problem space. It's literally what the business does. In our restaurant reservation example, we identified subdomains like Booking & Reservations, Restaurant Catalog, Reviews, Authentication, and Notifications. These exist whether we build software or not (Please take a moment and think about how a booking restaurant system could work without computers)

A bounded context lives in the solution space. We make a design decision about which subdomains we want to bundle together to share the same model. You might put multiple subdomains in one bounded context. Or you might split a single subdomain across multiple bounded contexts. It depends on what makes sense for your situation.

Now, what do I mean by "model" here? This trips up a lot of people. A model isn't just some data structures or a bunch of classes you wrote. A model represents the behaviors you want to provide for a specific part of your domain within a bounded context. It includes the business rules, operations, and constraints that define how that piece of the domain actually works.

Take our restaurant reservation system. The Booking & Reservations bounded context might handle behaviors like "reserve a table" and "cancel a reservation". The Restaurant Catalog bounded context provides different behaviors like "search restaurants" and "manage restaurant information".

Here's something interesting. Each bounded context maintains its own model with its own language. The same concept can mean totally different things depending on which context you're in.

Take "User" for example. In an Authentication Context, a User represents someone's login identity. Email, password, security roles, that sort of thing. But in the Booking Context, that same person becomes a Guest with a reservation history. In the Reviews Context, they become a Reviewer with a review history and credibility score. Same person, completely different meanings depending on the context.

## The Integration Challenge

In our previous post, I got pretty enthusiastic about the benefits of splitting our domain into smaller subdomains. We got better modularity. Clearer team responsibilities. Focused development efforts. Improved scalability. These benefits drove the whole microservices movement. The promise of autonomous services that can evolve independently sounds attractive.

But here's the thing nobody talks about upfront. There's a fundamental cost that comes with this separation. That cost is integration complexity.

When we split up what used to be a monolithic domain into multiple bounded contexts, we lose something valuable. The simplicity of everything living in the same process. But the bounded contexts still need to work together. Their models need to interact across context boundaries to deliver complete business functionality. You can't complete a restaurant reservation without accessing restaurant information, user authentication, and notification services.

If we mess up the division, we end up with what people call a "distributed monolith". Services that are physically separated but still tightly coupled logically. This creates all kinds of headaches. Due to the fact that each change needs coordinated updates across services, teams lose autonomy despite having physical boundaries between the services.

This is exactly why context mapping becomes so critical. It's our strategic approach to managing these integration challenges before they turn into technical disasters.

I've heard this conversation countless times in development teams:

* "Oh, we need some data provided by service Z"
    
* "It's not a problem, we can provide you a REST API endpoint"
    

Don't get me wrong. Providing an API isn't bad. But it usually leads to a situation where every service calls every other service. Both involved services expose APIs for each other. We propagate tons of data. We don't think about the source of truth. We ignore temporal coupling. Before you know it, you have a mess of interconnected services that are impossible to change independently.

## Strategy First, Technology Second

Context mapping is about identifying and defining the relationships between bounded contexts. Here's the key insight: context mapping comes before we start thinking about REST APIs, message brokers, or event streams.

Think of the context map like an architectural blueprint. It answers questions like: Who depends on whom? What kind of relationship should exist between these contexts? How should changes flow through the system? Once we make these strategic decisions, then we can pick the right technical patterns to implement them.

## Context Mapping Patterns

Strategic DDD gives us a vocabulary of well-established patterns. These patterns describe relationships between bounded contexts. Understanding these patterns is like learning the language of system integration.

### Understanding Upstream and Downstream

Most context relationships aren't symmetrical. There's usually an upstream and a downstream side. The upstream context provides the contract and has more control over the relationship. Think about an authentication service exposing a REST API. Or a booking service publishing "ReservationConfirmed" events.

The downstream context needs to adapt to the upstream interface. It has less control but more flexibility in how it uses the upstream data. Examples would be a booking service calling an authentication API. Or a notification service subscribing to reservation events.

Some patterns, like Shared Kernel, are symmetrical. Both contexts share equal responsibility and control over the shared elements.

### Open Host Service (OHS)

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1755773693388/f36fc4e7-6d18-4717-801a-bf38d30fe572.png align="center")

This pattern shows up when a bounded context provides a public API for mass consumption. The team maintaining the service is always upstream. They define the interface that everyone else has to integrate with. There's typically one well-documented interface that all consumers need to adapt to. Authentication services, notification platforms, and restaurant catalogs often follow this pattern.

### Published Language

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1755773705823/71bd3a15-27bd-4bab-8b61-55f1c0d3d44c.png align="center")

Open Host service is usually used with a Published Language. This pattern establishes a well-documented, shared language between bounded contexts. It's similar to the ubiquitous language concept. But where ubiquitous language applies within a bounded context, published language serves as the contract of a bounded context. Systems translate into and out of this published language. This reduces integration complexity when multiple systems need to communicate. It's often combined with open host services and acts like a reverse anti-corruption layer.

Well-documented examples include OpenAPI specifications for REST APIs or AsyncAPI specs for event-driven communication. These specifications define the exact structure, data types, and contracts that all integrating systems must follow.

### Conformist

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1755773720437/2c4b8bbb-94a6-42d1-bec9-70671b0904a1.png align="center")

Here, the downstream context just accepts whatever model the upstream provides. No transformation. The downstream context simply takes whatever the upstream gives them. This might seem like the easy choice, but be careful. It can create tight coupling that you don't need. Your downstream model might get polluted with concepts that don't really belong there. This makes changes more difficult than they should be.

### Anti-Corruption Layer (ACL)

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1755773778840/cbbc8dac-2430-429c-8fdb-2b99e4c8edce.png align="center")

This is the defensive approach. The downstream context creates a translation layer. This layer transforms the upstream model into something that fits its domain. It's more work upfront, but it protects your model from upstream changes. It keeps your domain clean. When the upstream API changes, you only need to update the translation layer.

### Shared Kernel

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1755773802659/c82613f1-af7c-404b-8a4c-76756f433c93.png align="center")

Two or more teams physically share a subset of the domain model. Usually through shared code, database schemas, or data structures. This sounds efficient, but it often becomes toxic. Changes require coordination between all teams. You end up with "god objects" that try to serve everyone. You lose the independence that bounded contexts should provide. Use this pattern very sparingly.

### Partnership

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1755773845374/cfa0ca0e-2d01-4e36-96f3-2e57d54f3d40.png align="center")

This is more about organization than technology. Teams that depend on each other coordinate their development schedules. They share responsibility for delivery success. It works when teams truly need to move in lockstep. But it can become a bottleneck for independent evolution.

### Customer-Supplier

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1755773860697/8f190294-a68e-4de1-a216-811c24a04756.png align="center")

The downstream team has some influence over the upstream team's roadmap. Requirements from the downstream factor into upstream planning. But the upstream team maintains final control over the interface. This creates a more collaborative relationship than pure conformist patterns.

### Separate Ways

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1755773923012/bde492fb-161d-44ab-bee7-8442b69da6be.png align="center")

Sometimes the best integration is no integration at all. Bounded contexts evolve completely independently with no connection between them. This works particularly well for MVP scenarios. You can defer integration complexity until you validate the core concepts.

### Big Ball of Mud (Anti-Pattern)

This is what happens when you ignore context mapping entirely. Everything becomes connected to everything else. Context boundaries disappear. Any change affects multiple parts of the system. Teams constantly fight over shared code. The ubiquitous language becomes meaningless. Development velocity grinds to a halt. It's the distributed monolith taken to its logical extreme.

## Some Rules of Thumb for Context Relationships

Context mapping requires careful consideration of your specific domain. But some useful guidelines can help with your decisions.

Generic and stable contexts should generally be upstream. Authentication, notifications, and payment systems typically don't change frequently. They can provide reliable contracts that other systems depend on. This helps minimize the impact of changes across your system.

Core subdomains often work best as downstream contexts. They need flexibility to evolve rapidly in response to business changes. When your core domain is downstream, changes in the core don't ripple through other subsystems. This allows your most important business logic to evolve without creating widespread impact.

Team dynamics also matter. Stronger or larger teams often naturally become upstream providers. Smaller teams adapt as downstream consumers. Understanding these organizational realities helps predict which relationship patterns will be sustainable over time.

## Applying Context Mapping to Our Restaurant System

Let's examine two key relationships in our restaurant reservation system. We'll explore different integration options. Rather than getting lost in theory, let's see how these choices play out in practice.

I'll provide pseudocode examples for better visualization, but remember - this part is about strategic relationship decisions, not implementation. Implementation should be derived from these strategic decisions, not the other way around.

### Booking Context and Notifications Context

You can implement the relationship between booking and notifications in two fundamentally different ways. The choice has significant implications for system stability.

#### Option 1: Booking Upstream

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1755774317612/492e53e5-5348-424a-8ddc-88ea4df1280f.png align="center")

The Booking context publishes the `ReservationConfirmed` event, while the Notification context consumes the message and transforms it into a local model.

```python
# Booking Context (Upstream)
class BookingService:
    def confirm_reservation(self, reservation_id):
        # Business logic
        self.event_publisher.publish(
            "ReservationConfirmed", 
            {
                "reservation_id": reservation_id,
                "guest_email": "guest@example.com",
                "restaurant_name": "Pizza Palace",
                "date": "2024-01-15",
                "time": "19:00"
            }
        )

# Notifications Context (Downstream)  
class NotificationService:
    def handle_reservation_confirmed(self, event_data):
        # Must know about reservation structure
        self.send_email(
            event_data["guest_email"],
            f"Reservation confirmed at {event_data['restaurant_name']}"
        )
    
    def handle_review_posted(self, event_data):
        # Needs new handler for each event type
        pass
```

This approach makes the Notifications Context unstable. Every time we want to send a new type of notification, the Notifications Context needs to change to handle the new event.

#### Option 2: Notifications Upstream

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1755774479818/8363abdb-6f3a-41a0-b8f3-58e72355ac3b.png align="center")

The Notification context exposes a stable API that is used by the Booking context when a reservation is made.

```python
# Notifications Context (Upstream)
class NotificationService:
    def send_notification(self, user_id, message, channel="email"):
        # Generic, stable interface
        return {"status": "sent", "notification_id": "123"}

# Booking Context (Downstream)
class BookingService:
    def confirm_reservation(self, reservation_id):
        # Business logic
        self.notification_service.send_notification(
            user_id="guest_123",
            message="Your reservation at Pizza Palace is confirmed",
            channel="email"
        )
```

The choice between these approaches depends on your architectural goals. If you want notifications to be truly generic and stable, make it upstream. If you have a limited set of notification types that don't change often, the event-driven approach might be simpler.

### Booking Context and Authentication Context

The relationship between booking and authentication contexts shows us three different integration approaches. Each has distinct trade-offs.

#### Option 1: Shared Kernel

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1755774714263/2065c47b-873f-4f6f-9ddf-1ff184bde5f2.png align="center")

Both use the same `User` class shared as a library.

```python
# Shared UserModel Library
class User:
    def __init__(self, user_id, email, password_hash, name, 
                 dietary_preferences, loyalty_level, registration_date):
        self.user_id = user_id
        self.email = email
        self.password_hash = password_hash
        self.name = name
        self.dietary_preferences = dietary_preferences  # Booking needs this?
        self.loyalty_level = loyalty_level              # Auth needs this?
        self.registration_date = registration_date      # Both need this?
    
    def verify_password(self, password): pass
    def get_booking_preferences(self): pass
    def calculate_loyalty_discount(self): pass  # Mixed responsibilities
```

This seems efficient, but it becomes a nightmare. When you change something in the library, all services are affected. The User object accumulates features needed by different services.

#### Option 2: Booking Conformist

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1755774725602/a5e52a0d-bde3-4632-935b-f7a05c847e39.png align="center")

The Booking context uses the Authentication context API to verify users and accept the model provided by the upstream context without any modifications.

```python
# Authentication Context (Upstream)
class UserService:
    def verify_user(self, user_id):
        return {
            "user_id": "123",
            "email": "guest@example.com", 
            "name": "John Doe",
            "dietary_preferences": ["vegetarian"],
            "loyalty_level": "gold",
            "registration_date": "2023-01-15",
            "last_login": "2024-01-10",
            "security_role": "customer"
        }

# Booking Context (Downstream) - stores everything
class BookingService:
    def verify_guest(self, user_id):
        user_data = self.user_service.verify_user(user_id)
        # Stores ALL data, even what booking doesn't need
        guest = Guest(**user_data)  # Pollution!
        return guest
```

Your booking context gets polluted with user information it doesn't actually need. You're storing email addresses, dietary preferences, and loyalty levels because that's what the upstream service provides.

#### Option 3: Booking implements ACL

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1755774740390/35207afd-34ff-405b-9af9-72fb491ee98f.png align="center")

The Booking context uses the Authentication context API to verify users and transform the model provided by the upstream context.

```python
# Authentication Context (Upstream)  
class UserService:
    def verify_user(self, user_id):
        return {
            "user_id": "123",
            "email": "guest@example.com",
            "name": "John Doe", 
            "dietary_preferences": ["vegetarian"],
            "loyalty_level": "gold",
            "registration_date": "2023-01-15",
            "last_login": "2024-01-10",
            "security_role": "customer"
        }

# Booking Context (Downstream) with ACL
class Guest:  # Domain-specific model
    def __init__(self, guest_id, email, is_authenticated):
        self.guest_id = guest_id
        self.email = email
        self.is_authenticated = is_authenticated

class UserAdapter:  # Anti-Corruption Layer
    def __init__(self, user_service):
        self.user_service = user_service
    
    def verify_guest(self, user_id):
        user_data = self.user_service.verify_user(user_id)
        # Transform to domain-specific model
        return Guest(
            guest_id=user_data["user_id"],
            email=user_data["email"], 
            is_authenticated=True
        )

class BookingService:
    def __init__(self, user_adapter):
        self.user_adapter = user_adapter
    
    def create_reservation(self, user_id):
        guest = self.user_adapter.verify_guest(user_id)
```

This requires more development effort upfront. You need to write and maintain the transformation logic. But it keeps your domain model clean and isolates you from upstream changes. The ACL approach aligns better with DDD principles because it preserves the distinct meaning of users in different contexts.

## Wrapping Up

Context mapping takes all those theoretical benefits we talked about in subdomain identification and turns them into real architectural decisions. When you understand these relationship patterns, you can design integration strategies that preserve the autonomy and flexibility you were after in the first place.

Here's the main takeaway: context mapping is strategic work that should drive your technical choices, not the other way around. Don't jump straight to REST APIs or message queues. Figure out the relationship between your bounded contexts first.

These decisions matter more than you might think. They affect how maintainable your system becomes, whether teams can work independently, and how fast you can deliver features. Get them right early and you'll thank yourself later as your system grows and gets more complex.