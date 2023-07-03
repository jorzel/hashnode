---
title: "Strategic Domain-Driven Design: Subdomains"
seoTitle: "Strategic DDD: Subdomains identification"
seoDescription: "Understanding what our business domain consists of and what are relationships between these parts are is crucial as it enables effective organization"
datePublished: Mon Jul 03 2023 07:53:19 GMT+0000 (Coordinated Universal Time)
cuid: cljmkdq71000i0al4bw4d3xc1
slug: strategic-domain-driven-design-subdomains-identification
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1688370651938/f2dbd39e-93f6-4952-8c0e-369b8fb99879.jpeg
tags: software-development, programming, software-architecture, ddd, domain-driven-design

---

## Strategic Domain-Driven Design Significance

A business domain refers to a specific area or aspect of a business or organization's operations, activities, and expertise. It represents a distinct sphere of knowledge, processes, and rules that govern the organization's functions and activities within a particular industry or market. Understanding what our business domain consists of and what are relationships between these parts are is crucial as it enables effective organization, modularization, and prioritization of development efforts, resulting in improved system design and alignment with business needs.

In the context of tackling a business domain, the strategic Domain Driven-Design (DDD) approach involves the following key aspects:

1. Subdomains Identification: The first step is to identify the distinct subdomains within the larger business domain. These subdomains represent distinct aspects of the business and encapsulate specific knowledge and functionality.
    
2. Bounded Contexts: Once the subdomains are identified, the next step is to establish bounded contexts. Bounded contexts define clear boundaries around a subdomain, where a specific model, language, and set of rules apply.
    
3. Context Mapping: It involves identifying the integration points, shared concepts, and collaborations between different bounded contexts. Context mapping helps determine how the bounded contexts communicate, exchange data, and maintain consistency, allowing for a holistic view of the system.
    

Identifying the core subdomain is one of the key benefits of strategic DDD. By analyzing the problem domain and identifying the core subdomain - the part of the business domain that provides the most value and differentiation- we can focus our efforts and resources where they matter most. Neglecting the strategic DDD step and relying solely on ad hoc decisions can lead to a system design that lacks a clear understanding of the core subdomain. This can result in a software solution that put a lot of effort into initiatives that do not provide a return on investment. Strategic DDD helps us prioritize our design efforts and ensure that the core subdomain is properly modeled, encapsulated, and protected from unnecessary complexity and dependencies. By dedicating more attention to the core subdomain, we can create a more robust and focused software solution. Furthermore, strategic DDD allows us to define bounded contexts, which are explicit boundaries that help manage the complexity of large-scale systems.

While tactical DDD patterns like factories, repositories, and aggregates are important for handling specific technical and business challenges, they are most effective when applied within the context of a well-defined strategic DDD approach. Without the strategic foundation, the tactical patterns implemented within a bounded context may not be utilized optimally, and the system can suffer from inconsistencies and design issues.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1688297612010/03bfdd23-9fd4-41c4-8a91-e45bb9d9ac5b.jpeg align="center")

By embracing strategic DDD, we gain a holistic understanding of the problem domain, identify the core subdomain, define bounded contexts, and ensure a well-aligned software design. This leads to a more maintainable, scalable, and adaptable system that can better support the evolving needs.

In this post, I would like to take a business domain problem: table reservations in restaurants, and consider how can we split the domain into subdomains and identify the core subdomain. Context mapping and bounded contexts determination are really broad topics and I will tackle them separately in future posts.

## Why do we need subdomains?

The first question that you may ask is why we want to split a domain into several smaller ones. In the introduction, we mentioned the most important reason: finding a core subdomain let us concentrate our efforts on it because it is the most important aspect of our business activity. Here are a few other reasons why subdomains are worth doing.

### Ubiquitous Language

One of the key principles of DDD is the use of a common language, called the ubiquitous language, shared between domain experts and developers. Splitting a domain into subdomains allows you to tailor the ubiquitous language to each subdomain. This means that the terminology and concepts used within a subdomain can be different from those used in another subdomain, reflecting the unique language used by domain experts in each area.

### Differentiation of Responsibilities

Each subdomain typically has different requirements, behaviors, complexity, and responsibilities. By identifying and separating subdomains, you can assign specific teams or individuals to focus on other areas of the domain. This promotes a clear division of responsibilities, enhances team autonomy, and enables more effective collaboration between domain experts and developers. It also allows for more focused development efforts and better alignment with business needs.

### Modularity

Subdomains often represent areas of the domain that can evolve independently. By dividing the domain into smaller, cohesive subdomains, you can create modular components that can be developed, tested, and maintained more easily. This modularity enables you to make changes to specific parts of the system without affecting the entire domain, facilitating agility and adaptability as business requirements change over time. By assigning data ownership to subdomains, it becomes easier to maintain consistency and integrity within each subdomain (providing a single source of truth).

### Scalability and Performance

Splitting a large, monolithic domain into smaller subdomains can help improve scalability and performance. Each subdomain can be scaled independently based on its specific usage patterns and resource demands. This allows for better allocation of resources and optimization of performance within each subdomain, leading to more efficient and responsive systems.

There is also a plain reason why we would like to chop the business domain into smaller parts. Working with a subset of a problem usually decreases its complexity and makes concentrating on crucial issues much easier.

## How to find subdomains?

The question of whether subdomains are discovered or constructed is a matter of perspective and can be interpreted differently based on individual viewpoints.

From one perspective, subdomains can be seen as discovered entities. They are derived from the inherent nature of the business domain and its complexities. Domain experts play a significant role in discovering these subdomains as they possess deep knowledge and insights into the business's operations and activities.

On the other hand, subdomains can also be considered constructed or subjective categories. The process of identifying and defining subdomains involves human interpretation, abstraction, and categorization. It requires analyzing and breaking down the business domain into meaningful parts based on specific criteria, such as cohesion, independence, and business value.

Identifying subdomains in DDD is a challenging process and typically involves a combination of heuristics, guidelines, and analysis techniques rather than strict methods or tools.

Here are some commonly used approaches.

### Business Processes and System Features

Analyzing the business processes, workflows, and user interactions within the domain can help identify areas of distinct functionality that could be potential subdomains. We must consider what functionalities our system should deliver and whether we can categorize them into different subdomains.

### Teams/Departments division

Mapping the business domain to existing teams or departments can provide initial insights into potential subdomains. Different teams often have specialized knowledge and responsibilities, which can align with distinct subdomains within the overall system. While aligning subdomains with teams or department topologies can provide a good starting point, it's important to exercise caution and not let existing organizational structures dictate the entire domain split. The goal should be to design subdomains that align with the desired architecture and promote a cohesive, maintainable, and scalable system.

[Conway's Law](https://en.wikipedia.org/wiki/Conway%27s_law) states that the structure of a software system will reflect the communication patterns and organizational structure of the team(s) building it. In the context of subdomain design, this means that if you blindly follow the existing team or department boundaries, you may end up with subdomains that are not well-aligned with the desired architecture or business capabilities.

### Subdomain patterns and types

Identifying the core subdomain, which represents the most critical and differentiating aspects of the business, can help in identifying other subdomains. The core subdomain is the area that provides the most value and competitive advantage, and it should form a separate subdomain. If we see that some parts of our can be disruptive, it is a candidate for the core subdomain that we should be focused on. We sometimes notice that some common and non-specific capabilities like notifications, invoicing, authentication, reporting, etc. are important but do not differentiate our business. They are great candidates to be generic subdomains that we can buy or outsource, but they should not be our main concern. The parts of the business domain that are neither crucial nor common, should be classified as supportive subdomains. More about finding subdomain patterns you can find in this great [article](https://medium.com/nick-tune-tech-strategy-blog/core-domain-patterns-941f89446af5), authored by Nick Tune.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1688046546839/792c90e3-c3fb-4610-abe8-89679f969c27.png align="center")

### Big-Picture Event Storming

[Event Storming](https://en.wikipedia.org/wiki/Event_storming) is a collaborative modeling technique that helps visualize and understand complex business processes.

[Big Picture Event Storming](https://medium.com/@chatuev/big-picture-event-storming-7a1fe18ffabb) is an initial workshop-style technique used to gain a high-level understanding of the business domain. It involves stakeholders from various roles and departments and aims to capture the major business processes, events, and actors. This technique helps identify the pivotal events, core domains, and major subdomains of the system. It provides a holistic view of the business and helps uncover key insights that can guide the subsequent design and modeling efforts.

Engaging domain experts and stakeholders in discussions, workshops, and knowledge transfer sessions are crucial for understanding the intricacies of the domain. Their expertise and insights can provide valuable guidance in identifying and defining subdomains.

### Pivotal Events and Ubiquitous Language

Pivotal events are significant occurrences or interactions within the business domain that drive the behavior and flow of the system. Identifying pivotal events helps identify cohesive areas of functionality that can potentially become separate subdomains. Analyzing the language used by domain experts and stakeholders can reveal patterns and variations, indicators of separate subdomains. Some concepts change their meaning in different contexts and it can be an indicator that they could belong to different subdomains.

Each of the above techniques can help us with proper subdomain identification. It is not a comprehensive set of heuristics. Every idea that can deepen our knowledge about the business domain can be instrumental and helpful.

## Subdomains for our example business domain

In our case, we do not have an existing system or company organization that we can use to get a starting point. We do not have access to domain experts, so a real Big-Picture Event Storming session is not an option. However, it is a good idea to think about what our system should deliver. When considering the business domain of table reservations in a restaurant, some of the system functionalities you may want to provide could include:

1. Search: Allow users to search for restaurants based on criteria such as location, cuisine, price range, availability, and other relevant filters.
    
2. Visitor preferences: Allow users to provide a unique profile matching their culinary preferences.
    
3. Recommendations: Logic for generating personalized restaurant recommendations based on user preferences and history.
    
4. Restaurant Information: Display a list of restaurants that match the user's search criteria, providing essential information such as restaurant name, location, opening hours, ratings, and customer reviews.
    
5. Table Availability: Provide real-time information on table availability, allowing users to select a suitable date and time for their reservation.
    
6. Reservation Booking: Enable users to reserve a table by specifying the desired date, time, number of guests, and any special requirements.
    
7. Reservation Confirmation: Send confirmation notifications to users upon successful reservation, providing them with the necessary details and any additional instructions.
    
8. Reservation Management: Allow users to view, modify, or cancel their existing reservations within a specified timeframe.
    
9. Guest Management: Capture guest information (name, contact details) during the reservation process to ensure efficient communication and service.
    
10. Restaurant Notifications: Send notifications to restaurant staff regarding incoming reservations, cancellations, and any special requests or requirements from guests.
    
11. Reporting: Generate reports and analytics related to reservation patterns, occupancy rates, popular time slots, and other relevant metrics to help optimize restaurant operations.
    
12. Restaurant Reviews: Allow users to leave reviews and ratings for restaurants they have visited.
    
13. Review supervising: Implement moderation or content guidelines to ensure the quality and appropriateness of reviews.
    
14. Authentication: Validate the identity of a registered user.
    

The set of proposed features is not comprehensive. However, it is enough to design a system that needs to be split into several subdomains to maintain its complexity.

Let's try to find some pivotal events that can make the above split more narrow and a little bit less granular. In the business domain we are considering a table reservation and adding a review could be a pair of pivotal events. These events transform the meaning of the `User` concept in the business domain. When the `User` is searching through restaurants to find an open table, she could be modeled as a `Vistor` within a subdomain responsible for restaurants listing. When the `Visitor` reserves a table, she becomes a `Guest` that could belong to a different subdomain - Booking and Reservations. Adding a review transforms the `Guest` into a `Reviewer`. These three models can share the same `user_id` field but are characterized by different data and behaviors.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1688055800807/98568ff0-aadc-43c4-b960-b622c1d246e3.png align="center")

Based on our findings, the initial business domain division could look as follow.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1688330232123/ee387661-9945-45ab-bf27-6102d4f7fc31.jpeg align="center")

Booking and Reservations (Reservation Booking, Table Availability, Reservation Management, Guest Management) can be considered a **core (green)** subdomain. Here's why:

* Business Differentiation: it directly contributes to the unique value proposition and differentiation of the business.
    
* Business Criticality: bookings and reservations effectively are fundamental to the success and operation of the business.
    
* Complexity and Specialization: booking and reservation systems often involve complex processes, such as managing availability, handling conflicts, coordinating resources, and ensuring accurate scheduling. This complexity and the specific domain knowledge required to handle these intricacies indicate that it can be a core domain.
    
* Strategic Importance: booking and reservations may represent a significant portion of the business revenue or customer engagement. Therefore, investing in the design and optimization of the booking and reservation process becomes strategically important for the business.
    

In the context of the restaurant reservation system, Reviews (Adding Reviews, Reviews Supervision) functionality can be classified as a **supportive (blue)** subdomain. The primary purpose of the system is to facilitate table reservations and provide information about restaurants. The review functionality supports this primary purpose by allowing users to share their experiences and provide feedback on the dining establishments. Reviews can influence the decision-making process of other users when choosing a restaurant for their reservation.

The Restaurants Catalog (Search, Restaurant Info, Visitor Preferences, Recommendations) subdomain can be either core or supportive but it depends on the specific business and its priorities. However, it is more commonly considered a **supportive (blue)** subdomain rather than a core one. The purpose of the Restaurants Catalog subdomain is to provide users with information about restaurants, such as location, cuisine, ratings, reviews, and other relevant details. While this functionality is essential for users to find and choose a restaurant, it supports the core functionality of booking and reservations.

Authentication, notifications, and reporting can be found in many systems, so they should be classified as **generic** **(yellow)** subdomains. It indicates that they provide common functionalities that can be reused across different parts of the system or even in multiple systems within an organization.

While a domain represents the overall problem space or business context, subdomains can further be divided into smaller subdomains. We are not restricted to two layers (domains on the first layer and subdomains on the second layer), but the domain division can be hierarchical, resembling a tree-like structure.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1688330940715/fc80898a-b377-4b7d-b3f4-532639bc1292.jpeg align="center")

This hierarchy allows for a more granular and organized representation of the domain. That design is never completed and finished. There is a high probability that in the future some of the features that are only modules of existing subdomains would evolve into separated subdomains.

## Conclusion

Knowledge crunching, which involves diving deep into the problem domain, seeking and acquiring domain knowledge, and understanding the intricacies of subdomains, is crucial for project success, especially in the context of strategic DDD. It requires dedication and effort to delve deeply into the domain, but the benefits in terms of better software design, improved communication, and higher customer satisfaction make it well worth the effort. Developers who master this skill are better equipped to develop software solutions that not only meet the functional requirements but also align closely with the real needs of the business.

If you would like to read more about the topic, I strongly recommend:

* classical Eric Evans [Blue Book](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215/),
    
* [an article](https://vaadin.com/blog/ddd-part-1-strategic-domain-driven-design) written by Peter Holmstrom,
    
* [Patterns, Principles, and Practices of DDD](https://www.amazon.com/Patterns-Principles-Practices-Domain-Driven-Design/dp/1118714709) book.