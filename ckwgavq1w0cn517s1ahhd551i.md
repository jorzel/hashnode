## Flask MVT. Refactor to service layer

In this short post, I would like to show how we can improve the separation of concerns using the Service Layer pattern within the Model View Template approach.

## Model View Template
MVT is an application-building pattern, commonly used in frameworks like Flask (but also Django).
It consists of three separated components:
1. Model layer that is responsible for domain logic and persistence.
2. View that concerns handling HTTP requests. It is a place where application API is defined. The view orchestrates application logic by interacting with the model and executing requests to external services.
3. Template is a presentation layer (e.g. HTML templates) that handles the User Interface part.

![mvt.drawio.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1637924048990/_vyXOsAHI.png)

MVT approach often results in the following code smells:
- mixing responsibilities of handling HTTP requests (e.g. operating on query string params, cookies, headers, etc.) with application logic orchestration (strong coupling between a framework and business requirements).
- complicated test setup (always need an HTTP request to test application logic).
- mixing responsibilities of domain logic and persistence (infrastructure concern)

## Push your framework outside application logic
A solution that can ease the first two of the three mentioned smells is an introduction of a service layer (it can be a class but also a standalone function, depending on the specific case). It can decouple application logic from a framework we use (application features/use cases become agnostic of framework code). The view is now responsible for handling requests and executing service layer methods/functions, while application logic is now only a concern of a service layer. It allows us to focus on the application core at first, and implement API that enables entry to it afterward. Thanks to the separation, API does not necessarily need to be API supporting rendered templates but it can be CLI (Command Line Interface), REST API, GraphQL API, RPC, etc. The decision about API (and framework) can be deferred, but we still can develop our application features. 
API and application logic separation could also improve our tests. We could now write tests of the application core, covering logical paths extensively, while covering only the happy path for the API part.

![mvt_service.drawio.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1637918478947/8Uk5e6ZWI.png)

## Conclusion
This solution is not a silver bullet of course. We should consider it when our view endpoints are large and not trivial, and writing clean, readable code is challenging. 