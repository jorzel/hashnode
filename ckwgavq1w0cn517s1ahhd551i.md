## Flask MVT. Refactor to service layer

In this short post I would like to show how we can improve separation of concerns using Service Layer pattern within Model View Template approach.

## Model View Template
MVT is an application building pattern, commonly used in frameworks like Flask (but also Django).
It consists of three separated components:
1. Model layer that is responsible for domain logic and persistence.
2. View that concerns about handling http request. It is a place where application API is defined. The view orchestrate application logic by interaction with model and executing requests to external services.
3. Template is a presentation layer (e.g. html templates) that handles User Interface part.

![mvt.drawio.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1637924048990/_vyXOsAHI.png)

MVT approach often results in following code smells:
- mixing resposibilites of handling http request (e.g. operating on query string params, cookies, headers, etc.) with application logic orchestration (strong coupling between a framework and business requirements).
- complicated test setup (always need a http request to test application logic).
- mixing resposibilities of domain logic and persistence (infrastructure concern)

## Push your framework outside application logic
A solution that can ease first two of three mentioned smells is an introduction of a service layer (it can be class but also standalone function, depending on the specific case). It can decouple application logic from a framework we use (application features / use cases become agnostic of framework code). View is now resposible for handling requests and executing service layer methods / functions, while application logic is now only a concern of a service layer. It allows us to focus on application core at first, and implement API that enables entry to it afterwards. Thanks to the separation, API does not necessary need to be API supporting rendered templates but it can be CLI (Command Line Interface), REST API, GraphQL API, RPC, etc. The decision about API (and framework) can be deferred, but we still can develop our application features. 
API and application logic separation could also improve our tests. We could now write tests of application core, covering logical paths extensively, while covering only happy path for API part.

![mvt_service.drawio.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1637918478947/8Uk5e6ZWI.png)

## Conclusion
This solution is not a silver bullet of course. We should consider it when our view endpoints are large and not trivial, and writing clean, readable code is challenging. 