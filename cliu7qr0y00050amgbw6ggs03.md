---
title: "The Power of Conventions in Software Development"
seoTitle: "The Power of Conventions in Software Development"
seoDescription: "Conventions provide a framework for consistency and promotes shared understanding and predictable outcomes"
datePublished: Tue Jun 13 2023 11:41:58 GMT+0000 (Coordinated Universal Time)
cuid: cliu7qr0y00050amgbw6ggs03
slug: the-power-of-conventions-in-software-development
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1686652071089/881a0efe-ca28-4a6c-98c2-6c8e9644593d.jpeg
tags: software-development, programming, software-architecture, teamwork

---

## Introduction

A convention is a widely accepted and agreed-upon practice, rule, or standard that guides behavior, decision-making, or design within a specific context. It provides a framework for consistency and promotes shared understanding and predictable outcomes.

Conventions can play an important role in reducing cognitive load in software development. Cognitive load refers to the amount of mental effort and resources required to process information and perform a task. It is a measure of the cognitive burden placed on an individual's working memory. Cognitive load theory, proposed by John Sweller, distinguishes between three types of cognitive load:

1. **Intrinsic** cognitive load is the inherent complexity associated with the task itself. It is determined by the complexity of the content being learned or processed. Some tasks naturally require more mental effort due to their inherent difficulty, novelty, or the need to process multiple elements simultaneously.
    
2. **Extraneous** cognitive load refers to the cognitive burden that arises from factors external to the task or learning material. It is caused by poorly designed instructional materials, irrelevant or distracting information, complex navigation, or confusing presentation formats. When extraneous cognitive load is high, it hampers the learning or problem-solving process by diverting mental resources away from the task at hand.
    
3. **Germane** cognitive load is the cognitive effort that is directly related to the learning or problem-solving process and contributes to building long-term knowledge and skills. It represents the mental effort invested in understanding, organizing, and integrating new information into existing mental models or schemas. Germane cognitive load is considered desirable because it leads to meaningful learning and deeper understanding.
    

![What is Cognitive Load Theory? - by David Weller](https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fbucketeer-e05bbc84-baa3-437e-9518-adb32be77984.s3.amazonaws.com%2Fpublic%2Fimages%2F15c0ae1a-e57f-4cfd-9046-927a370d0bed_1050x700.png align="left")

Optimizing cognitive load involves managing the interplay between these three types. Minimizing extraneous cognitive load and focusing on germane cognitive load can help learners or problem solvers allocate their limited mental resources more effectively and enhance their performance and understanding.

By using conventions to establish consistent patterns and structures, developers and IT professionals can reduce the cognitive load required to process information and understand the system. Following conventions usually increase the germane cognitive load to some extent, as individuals may need to invest mental effort into learning and internalizing new conventions. However, once the conventions have been internalized, they can help to reduce extraneous cognitive load by making it easier to understand and navigate the system.

## Convention-driven solutions

Convention-driven solutions, like white-box systems, provide a clear and accessible design, follow established patterns, and reduce extraneous cognitive load by providing a shared understanding. Ad hoc solutions on the other hand, akin to black or grey box systems, may lack clear design and conventions, requiring reverse engineering and imposing a higher cognitive load on developers.

Convention-based solutions are designed and developed based on established conventions, best practices, and agreed-upon standards. These conventions provide a structured approach to problem-solving and solution design. These solutions typically have a well-thought-out design that follows established patterns and principles. The design decisions are made consciously, considering factors such as modularity, scalability, maintainability, and reusability. The design is often shared and understood by the development team, enabling effective collaboration.

Since convention-driven solutions adhere to established conventions, they exhibit a level of predictability. Developers familiar with the conventions can quickly understand how the solution is structured, how different components interact, and where to find specific functionality. This predictability makes it easier to maintain and enhance the solution over time. Convention-driven solutions help reduce extraneous cognitive load by providing a clear structure and shared understanding. Developers don't need to spend excessive mental effort on deciphering the design decisions or reverse-engineering the solution. Instead, they can focus on understanding and implementing the specific requirements within the established framework.

Ad hoc solutions, on the other hand, are often created without following explicit conventions or predefined design patterns. They are built to address immediate needs or specific requirements without a comprehensive, systematic approach. Ad hoc solutions may lack a well-defined design. They are often built on an as-needed basis, without considering long-term scalability or maintainability. The design decisions may be driven by immediate concerns rather than a holistic approach.

Ad hoc solutions may exhibit lower predictability compared to convention-driven solutions. The lack of established patterns can make it challenging for developers to understand the system's structure and behavior. Reverse engineering and deciphering the decision process may be required, leading to increased cognitive load. Developers need to spend more time and effort understanding the solution's internals, making it harder to maintain, modify, or enhance the system over time. Ad hoc solutions may require additional cognitive effort to navigate and comprehend the codebase.

For a better illustration of the difference between the above approaches imagine the task of putting clothes into the chest of drawers.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1686648677419/e491e1c9-fc60-446e-893b-5c38ded22604.png align="center")

In the convention-based solution, there is a predefined set of conventions or rules for organizing clothes in the drawers. These conventions could be based on factors like clothing type, color, or season. For example, you might have a convention where socks are always placed in one specific drawer, shirts in another, and so on.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1686649536254/eeadd410-b5bc-4b96-a5f7-0d999171c564.png align="center")

In contrast, an ad hoc solution involves putting clothes into drawers without any specific design or conventions. Each clothing item is placed in a drawer without a predetermined system.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1686650409051/57f987d4-ea8d-4765-b6f1-0f30aa632403.png align="center")

While this approach may still get the job done, it presents challenges:

* Lack of Organization: Without conventions, the organization of clothes becomes haphazard. Clothes might end up mixed together, making it difficult to locate specific items quickly.
    
* Unpredictability: With an ad hoc approach, it becomes harder to predict where a particular item of clothing is stored. You would need to search through various drawers to find what you are looking for, increasing the time and cognitive effort required.
    
* Inefficiency: Retrieving specific clothing items from an ad hoc system can be more time-consuming and frustrating. You may need to check multiple drawers and possibly rearrange clothes each time to locate what you need.
    

## Conventions in software development

By following conventions team members can develop a shared mental model of how the application is structured and how components interact with each other. This can help to reduce cognitive load by providing a clear understanding of the solution and how it works.

Furthermore, conventions can help to establish a shared understanding of the problem domain itself. By using established terminology and practices, team members can develop a common mental model of the problem domain, which can help to identify potential issues and develop effective solutions.

In the following sections, I would like to present some areas where solutions based on conventions can bring a lot of benefits.

### Naming in code

The main reason for obscurity in the code that dramatically increases cognitive load is the bad naming of objects. Consistency and precision in wording are essential for writing clean and understandable code:

* **Precise** names contribute to the readability and maintainability of code. When variables, functions, or classes are named based on their purpose or responsibility, it becomes easier for other developers (including your future self) to understand their intent and use them correctly. Precise names reduce confusion and make the code more self-explanatory.
    
* **Consistency** in naming is crucial for creating a cohesive and unified codebase. For example, consistency in verb usage is important for conveying intent and avoiding ambiguity. Choosing appropriate verbs helps communicate the intended action clearly. It is really confusing when we use different verbs like "edit" "modify", "update" or "change" interchangeably for the same operation. Using the same name consistently for the same concept improves clarity and reduces the cognitive load when reading and working with the code. Inconsistent naming can lead to confusion and make it more challenging to understand and maintain the codebase.
    

Following naming conventions contributes to creating self-documenting code. When names are precise and consistent, the code becomes more expressive and easier to understand without relying heavily on additional comments or documentation. Clear and meaningful names can serve as effective documentation in themselves.

### Codebase structure

"Screaming Architecture" is a concept [coined by Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2011/09/30/Screaming-Architecture.html) (Uncle Bob) to emphasize the importance of designing software architectures that clearly and loudly express the intent and domain of the system.

The idea behind Screaming Architecture is to create a clear separation of concerns and to make the architecture of the system "scream" the important design decisions and business rules. The architecture should be structured in a way that the high-level policy and domain-specific elements are easily identifiable and distinguishable from the low-level implementation details.

Screaming Architecture encourages the use of clean and well-defined boundaries between different architectural layers and components. This separation allows for flexibility, maintainability, and testability, as well as enables an easier understanding of the system's design and intent.

By following conventions, developers can ensure that the architecture "screams" the key design decisions and makes the intent of the system clear to anyone working on it.

As an example, I would like to describe the convention of separating a backend-related project into four layers that align with common architectural patterns, such as Domain-Driven Design (DDD) and Clean Architecture. It offers several benefits and can enhance the maintainability, testability, and scalability of a system. Here are some thoughts on each layer:

1. **API Layer**: This layer serves as the entry point for external communication with the system. It provides an interface for clients to interact with the backend via various transport protocols. Separating this layer helps decouple the presentation concerns from the rest of the system and enables flexibility in adapting to different communication channels.
    
2. **Infrastructure Layer**: The infrastructure layer encapsulates the implementation of external dependencies such as databases, message brokers, and external service clients. It also implements the interfaces defined in the application or domain layer, allowing them to interact with external systems. This separation facilitates the management of infrastructure concerns and keeps the core business logic independent of specific technical details.
    
3. **Application Layer**: The application layer focuses on business orchestration and use case implementation. It encapsulates the high-level business logic and coordinates the interactions between different domain objects. This layer should be free from infrastructure concerns and serve as the primary entry point for understanding the system's main business use cases.
    
4. **Domain Layer**: The domain layer represents the core of the system and contains the business concepts, rules, and domain models. It captures the essential business logic and ensures the integrity and consistency of the domain. By isolating the domain from other layers, it becomes easier to reason about and maintain the business rules independently of technical details.
    

![](https://raw.githubusercontent.com/jorzel/opentable/master/clean.png align="left")

The above convention helps to establish clear boundaries between different aspects of the system and promotes the separation of concerns. It put some restrictions on the developer where she can place a new code. It allows for modular development, easier testing, and scalability. The application and domain layers, in particular, provide a focused view of the system's business use cases and domain concepts, aiding in the understanding of its primary purpose.

### Commit messages

Conventions play an important role when making commits to version control systems. Commit messages are crucial for documenting the changes made to a codebase over time, and following conventions when writing commit messages can greatly improve collaboration, code maintenance, and project management. Here are a few conventions to consider:

1. Clear and concise messages: Commit messages should be brief but descriptive, summarizing the purpose or intent of the changes made.
    
2. Use of imperative mood: It is a common convention to write commit messages in the imperative mood, such as "Add feature X" or "Fix issue Y." This helps maintain consistency and readability across commit messages.
    
3. Reference relevant issues or tickets: If your project uses an issue tracking system or ticketing system, consider referencing the relevant issue or ticket number in the commit message. This helps link commits to specific tasks or bug reports, providing additional context.
    
4. Separate concerns with a blank line: For complex commits that address multiple concerns or changes, it is beneficial to separate them with a blank line in the commit message.
    

Here is [the article](https://cbea.ms/git-commit/) that provides even more rules to take into account when writing commits. It's important to establish and follow conventions that are agreed upon within your team or organization.

### Service instrumentation

Conventions play a significant role in the service instrumentation field, especially when it comes to logging and metrics.

In the context of logging, using structured logging with consistent key names can greatly enhance the searchability and interpretability of logs. By adhering to conventions for key names (e.g., using "user\_id" consistently instead of using multiple variations like "user," "owner," "client"), it becomes easier to search, filter, and analyze logs. Consistency in key names also improves the overall readability and maintainability of log data, making it more efficient to identify and troubleshoot issues.

Similarly, in the realm of metrics, having a well-defined convention for metric naming is crucial for effective monitoring and analysis. Prefix conventions can be particularly helpful in providing clear and consistent context. By including the organization slug, service name, and metric name as part of the metric naming convention, it becomes easier to identify and categorize metrics. This convention helps maintain clarity and consistency across various services and metrics within an organization.

Conventions in service instrumentation promote uniformity, standardization, and ease of use across teams and systems. They enable efficient log analysis, facilitate troubleshooting, and provide a solid foundation for monitoring and observability practices. By following conventions in service instrumentation, organizations can enhance their ability to understand and optimize the performance and behavior of their services.

## Conclusion

Conventions can serve as a type of contract between team members that we would solve problems in a predictable and designed way.

The important point is that breaking a convention may not have immediately noticeable effects ("it's not a big deal"), and the consequences can be deferred, which can lead to decreased maintainability and increased surprises for others working on the solution. The solution may continue to function correctly, giving a false impression that breaking the convention is inconsequential. However, over time, the deferred consequences start to emerge, making the solution harder to understand and maintain.

It is important to emphasize the value of maintaining conventions and fostering a culture of adherence to established standards. This can be achieved through code reviews, team discussions, and ongoing education about the benefits of conventions in promoting maintainability and reducing surprises. Encouraging communication and collaboration within the team can help ensure that everyone understands and agrees on the conventions in place, increasing the likelihood of their continued adherence.