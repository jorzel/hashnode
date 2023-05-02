---
title: "Step-by-Step Guide to Safely Introducing High-Risk Changes in Your Software System"
seoTitle: "Step-by-Step Guide to Introducing High-Risk Changes in Your Software"
seoDescription: "When introducing a change to a software system, it's essential to do so in a way that minimizes the risk of unintended consequences"
datePublished: Tue May 02 2023 08:17:04 GMT+0000 (Coordinated Universal Time)
cuid: clh5zxgig001b0am8el3n1ck2
slug: step-by-step-guide-to-safely-introducing-high-risk-changes-in-your-software-system
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/DT71hAJoIJM/upload/b3fdbc2a7290e508826624e9b7ed39bb.jpeg
tags: software-development, programming, software-engineering, risk-management

---

## Introduction

When introducing a change to a software system, it's essential to do so in a way that minimizes the risk of unintended consequences or negative impact on users. Even if we have a new solution tested, we can still be afraid of deploying it at scale because of the high cost of possible errors. It's important to plan and coordinate any changes to a software system carefully, and to communicate clearly with users about any potential impacts or downtime.

In this post, I would like to introduce the following concepts:

* parallel models,
    
* shadow deployment,
    
* feature flags,
    
* gradual rollout,
    

and present how combining these techniques can help minimize the risk of making a troublesome change.

## Parallel models

Parallel models involve running two versions of the system in parallel, with a small subset of users being directed to the new version. This allows for testing and validation of the new version without risking widespread impact on users. Once the new version has been thoroughly tested and validated, more users can be gradually transitioned to the new version.

A new model is just a new version of the software part we want to change. It can be a new feature, performance refactoring, architecture refactoring, etc.

To better understand it, please assume that we have a running system that is using the current solution behind `bar` interface.

```python
def bar(user_id: str, *args, **kwargs) -> Any:
    # current implementation
    return result
          
def run(user_id: str) -> Any:
    return bar(user_id)
```

Now, we want to refactor `bar` implementation. Due to the fact, it is a high-risk change, we decide to introduce a totally new model (function) that will replace `bar` implementation in the future, instead of just rewriting `bar` code in place.

```python
def improved_bar(user_id: str, *args, **kwargs) -> Any:
   # new solution implementation
   return result
```

As a result, we have two different pieces of code that are meant to do the same job and should be interchangeable from the functional point of view.

## Shadow deployment

At the very beginning, we will not replace `bar` function by `improved_bar`. We want to execute both models quasi-simultaneously but take results only from the legacy one. It is a technique known as shadow deployment, where a new implementation is introduced only to log results.

```python
import logging

def bar(user_id: str, *args, **kwargs) -> Any:
   # current solution implementation
   return result

def improved_bar(user_id: str, *args, **kwargs) -> Any:
   # new solution implementation
   return result

def run(user_id: str) -> Any:
    legacy_results = bar(user_id)
    logger.info("Legacy model results: ", legacy_results)
    new_results = improved_bar(user_id)
    logger.info("New model results: ", new_results
    if new_results != legacy_results:
        logger.warning("New and legacy models results mismatch")
    return legacy_results
```

If we are interested in time comparison between models, we can decorate both functions with a metrics handler that would store a function latency (see [this post](https://jorzel.hashnode.dev/how-to-use-prometheus-for-web-application-monitoring) to check how to collect metrics using Prometheus).

Running parallel models in a shadow mode could be tricky and challenging when our code modifies the database state or publish events to the message broker. In that case, we must be sure that side effects coming from running parallel models would not corrupt our system logic.

In some situations, we cannot allow running parallel models due to performance issues (a new model would consume some request time). In that case, we could consider executing a new model with only a small fraction of requests or call the new model asynchronously if possible.

## Feature flags

When we are satisfied with the logs from the shadowing experiment, we can start thinking about using the new model "seriously". Feature flags are a technique for selectively enabling or disabling specific features of the system. By enabling the feature for a small subset of users initially, any issues or bugs can be quickly identified and addressed before wider rollout. We have probably several options for how it can be done:

A. A simple boolean flag that determines which solution (legacy vs new) would be used

```python
import os
import logging

is_enabled = bool(os.environ.get('IS_ENABLED', False))

def should_use_improved_bar() -> bool:
    return is_enabled

def run(user_id: str) -> Any:
    if should_use_improved_bar():
        logger.info("Use new model for {user_id=}")
        return improved_bar(user_id)
    return bar(user_id)
```

B. Every n-th request should hit a new solution

```python
import os
import logging

nth = int(os.environ.get('NTH_REQUEST', 100))

counter = 0

def should_use_improved_bar(
   user_id: str, counter: int, nth: int
) -> bool:
    return counter % nth == 0

def run(user_id: str) -> Any:
    if should_use_improved_bar(user_id, counter, nth):
        logger.info("Use new model for {user_id=}")
        return improved_bar(user_id)
    counter += 1
    return bar(user_id)
```

C. A stable fraction (percentage) of users should be routed to a new model

```python
import os
import hashlib
import logging

# range [0;1]
fraction = float(os.environ.get('USERS_FRACTION', 0)) 

def should_use_improved_bar(
    user_id: str, fraction: float
) -> bool:
    hash_hex = hashlib.md5(user_id).hexdigest()
    hash_int = int(hash_hex, 16)
    hash_reduced = hash_int % 100
    return hash_reduced < fraction * 100
          
def run(user_id: str) -> Any:
    if should_use_improved_bar(user_id, fraction):
        logger.info("Use new model for {user_id=}")
        return improved_bar(user_id)
    return bar(user_id)
```

In all of these implementations, we are using a simple `if` condition (based on results from `should_use_improved_bar`) function to route (or not) to the new solution.

## Gradual rollout

If we take B or C option from the previous section, we can manipulate the amount of traffic that is routed to the new solution. Gradual rollout involves slowly increasing the percentage of users who are exposed to the new version of the system. This can be done in stages, such as starting with 10% of users (or requests) and increasing by 10% increments every few hours or days. This allows for any issues or bugs to be identified and addressed before a larger percentage of users are affected. Even if some unexpected issues occur, it will not affect all users using the system.

For example, we can implement the following algorithm:

* Start with a small fraction of requests (let's say 1%) routed to the new solution.
    
* Monitor application logs and metrics to ensure that everything works fine. If something goes wrong, reduce the percentage of requests routed to the new solution or disable it completely.
    
* Gradually increase the fraction of requests routed to the new solution, monitoring logs and metrics at every stage.
    
* Once the new solution has been rolled out to 100% of users, disable the old solution
    

## Conclusion

We have shortly described how to make high-risk changes in software development more reliable and predictable. These techniques allow teams in a really simple way to validate changes in real production environments and making a step back without code modifications if things are not going well. Although not all changes have to be introduced with that level of carefulness, I believe that approach should be in all software developers' toolkits.