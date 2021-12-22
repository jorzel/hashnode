## GraphQL API and REST API. Mirror implementations in Python

## Introduction
REST is probably the most popular way to expose your application to the external world (e.g. as a backend for frontend or to establish communication protocol with other application / service). However, GraphQL is now getting more and more popular, and have became a strong competitor for REST. 
Nevertheless, the aim of this post is not to carry out a detailed comparison of advantages / disadvantages of these approaches, because there is a lot of stuff covering that topic. I would rather like to present how REST and GraphQL differ in implementation strategies. This post can be especially helpful for people who are familiar with REST pattern and wonder how to broaden interests in GraphQL technology.

## Application requirements
First of all, I create simple `flask` project with `Blueprint` functionality to provide REST API. In contrast to REST, that need separated HTTP endpoint for each business action, GraphQL expose only one HTTP endpoint both for queries and commands. There is a `flask_graphql` package that expose this endpoint with request syntax parsing provided. To built GraphQL schema I use `graphene` library and `sqlalchemy` to build simple ORM models. To present results of both APis I use  [`insomnia`](https://insomnia.rest/download)  that is a powerful REST HTTP client with GraphQL integration. 

```bash
pip install flask
pip install flask_graphql
pip install graphene<3
pip install sqlalchemy
```

## Simple query
We start from simple healthcheck feature that ensure us that our application setup is valid and both REST API and GraphQL API is exposed to the world. For REST we define simple HTTP endpoint within the blueprint.
```python
# api/rest.py
from flask import Blueprint,  jsonify

main = Blueprint("main", __name__)

@main.route("/")
@main.route("/up")
def up():
    return jsonify({"up": True})
```
Here is a query result for `GET` request in insomnia:
![rest_up.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1640120747886/gBd2dteav.png)

GraphQL is a strongly typed query language, so we must define `Schema` with root object `Query` that defines possible entry points into the GraphQL API. For each field of the schema we define a resolver function that determine the returned value (it usually use a database connection / session to determine what value should be returned).
```python
# api/graphql.py
import graphene

class Query(graphene.ObjectType):
    up = graphene.Boolean()

    def resolve_up(root, info, **kwargs):
        return True

schema = graphene.Schema(query=Query)
```
Both queries and commands in GraphQL are executed as `POST` requests. So in insomnia we choose `POST` request structured as GraphQL Query. 
![graphql_up.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1640120902863/xnwNgVrrv.png)


## Query real objects
We have our "hello world" working, so now we can turn to real objects quering. To do it, we define a simple data model of `Restaurant` using SQLAlchemy ORM.
```python
# models.py
from sqlalchemy import Column, Integer, String

from db import Base

class Restaurant(Base):
    __tablename__ = "restaurant"

    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String)
```

To query all `Restaurant` instances in REST we add new HTTP endpoint and `recruitment_serializer` to transform our model into JSON-like format.
```python
from flask import Blueprint, current_app, jsonify

from models import Restaurant

main = Blueprint("main", __name__)

def restaurant_serializer(restaurant: Restaurant) -> Dict[str, Any]:
    return {"id": restaurant.id, "name": restaurant.name}

@main.route("/restaurants", methods=["GET"])
def restaurants():
    session = current_app.session
    restaurants = [restaurant_serializer(r) for r in session.query(Restaurant)]
    return jsonify(restaurants)
```
In insomnia, we execute a `GET` request at `/restaurants` url:
![rest_restaurants.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1640121797362/wZH19DAOc.png)

For GraphQL we need to define `RestaurantNode` schema object. `RestaurantConnection` is an additional object that provide some useful features for working on collections (like results limiting, pagination, etc.).
```python
# api/graphql.py
import graphene
from models import Restaurant

class RestaurantNode(graphene.ObjectType):
    class Meta:
        interfaces = (graphene.relay.Node,)

    name = graphene.String()

class RestaurantConnection(graphene.Connection):
    class Meta:
        node = RestaurantNode

class Query(graphene.ObjectType):
    up = graphene.Boolean()
    restaurants = graphene.relay.ConnectionField(RestaurantConnection)

    def resolve_up(root, info, **kwargs):
        return True

    def resolve_restaurants(root, info, **kwargs):
        session = info.context["session"]
        return [RestaurantNode(id=r.id, name=r.name) for r in session.query(Restaurant)]

schema = graphene.Schema(
    query=Query, types=[RestaurantNode]
)
```
To query the list of restaurants we execute `POST` request and determine which fields we would like to get (we still have `up` field in our schema, but it is not necessary for us now, so we don't query it) using the same `/graphql` endpoint:
![graphql_restaurants.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1640121499402/JmhB1jKw-.png)

## Command
We have seen how to query objects using both REST and GraphQL. In the following section I would like to present how to modify data. To make it happen, the domain model was enriched by adding `Table`, `TableBooking` and `User` entities. 
```python
# models.py
from datetime import datetime
from typing import Optional

from sqlalchemy import Boolean, Column, DateTime, ForeignKey, Integer, String
from sqlalchemy.orm import relationship

from db import Base

class NoOpenTable(Exception):
    pass

class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True, autoincrement=True)
    email = Column(String, unique=True)
    table_bookings = relationship("TableBooking")

class Table(Base):
    __tablename__ = "table"

    id = Column(Integer, primary_key=True, autoincrement=True)
    max_persons = Column(Integer, nullable=False, default=2)
    is_open = Column(Boolean, default=True, nullable=False)
    restaurant_id = Column(ForeignKey("restaurant.id"))
    restaurant = relationship("Restaurant")

    def book(self):
        self.is_open = False

class Restaurant(Base):
    __tablename__ = "restaurant"

    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String)
    tables = relationship("Table", lazy="dynamic")

    def book_table(self, persons: int, user: User):
        table = self._get_open_table(persons)
        if not table:
            raise NoOpenTable()
        table.book()
        return TableBooking(user=user, restaurant=self, persons=persons)

    def _get_open_table(self, persons) -> Optional[Table]:
        return self.tables.filter(
            Table.is_open.is_(True), Table.max_persons >= persons
        ).first()

class TableBooking(Base):
    __tablename__ = "table_booking"

    id = Column(Integer, primary_key=True, autoincrement=True)
    booked_at = Column(DateTime, default=datetime.utcnow)
    user_id = Column(ForeignKey("user.id"), nullable=False)
    user = relationship("User")
    restaurant_id = Column(ForeignKey("restaurant.id"), nullable=False)
    restaurant = relationship("Restaurant")
    persons = Column(Integer, nullable=False, default=1)
```
Our command that modify the state of the domain model, will be booking a `Table` in given `Restaurant` by `User` identified by `email`. Both REST and GraphQL are clients for the same application logic, so we abstract it into a service layer function `book_restaurant_table` that is responsible for whole business action:
```python
from sqlalchemy.orm import Session
from models import Restaurant, TableBooking, User

def book_restaurant_table(
    session: Session, restaurant_id: int, user_email: str, persons: int
) -> TableBooking:
    user = session.query(User).filter_by(email=user_email).first()
    restaurant = session.query(Restaurant).get(restaurant_id)
    table_booking = restaurant.book_table(persons, user)
    session.add(table_booking)
    session.commit()
    return table_booking
```
REST to modify resources use `POST`, `PUT` or `DELETE` verbs. To add a new `TableBooking` instance as result of booking action, we implement new HTTP `POST` endpoint that handle the request and execute `book_restaurant_table` use case:
```python
from flask import Blueprint, current_app, jsonify, request
from service import book_restaurant_table

main = Blueprint("main", __name__)

@main.route("/bookings", methods=["POST"])
def bookings():
    session = current_app.session
    payload = request.get_json()
    _ = book_restaurant_table(
        session,
        restaurant_id=payload["restaurant_id"],
        user_email=payload["user_email"],
        persons=payload["persons"],
    )
    return jsonify({"isBooked": True}), 201
```
Performing a request `POST` in insomnia with input data result in response with 201 status code:

![rest_post.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1640175308499/zDdICBqpt.png)

In GraphQL mutation object must be define to carry out an operation that result in side effects (change of state). We define `BookRestaurantTable` and also declare it in the GraphQL schema:
```python
import graphene
from graphene.relay.node import from_global_id
from service import book_restaurant_table

class BookRestaurantTable(graphene.Mutation):
    class Arguments:
        restaurant_gid = graphene.ID(required=True)
        persons = graphene.Int(required=True)
        user_email = graphene.String(required=True)

    is_booked = graphene.Boolean()

    def mutate(self, info, restaurant_gid: str, persons: int, user_email: str):
        session = info.context["session"]
        _, restaurant_id = from_global_id(restaurant_gid)
        _ = book_restaurant_table(session, restaurant_id, user_email, persons)
        return BookRestaurantTable(is_booked=True)

class Mutation(graphene.ObjectType):
    book_restaurant_table = BookRestaurantTable.Field()

schema = graphene.Schema(
    query=Query, mutation=Mutation, types=[UserNode, RestaurantNode]
)
```
To execute a mutation we use the same insomnia setup, but the body of GraphQL request is a little bit different from query one. We use `mutation` key-word and pass camel case name of mutation declared here: `book_restaurant_table = BookRestaurantTable.Field()`. Response returns the result payload with 200 status code:

![graphql_mutation.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1640175553194/amZ_qcB29P.png)

## Parameterized queries
We showed simple query that fetch all `Restaurant` instances. However, we usually want to parameterize query by passing arguments, limiting results, making searches, etc. Like for mutation application logic can be agnostic of API that consume it, so we can implement it in service layer as `get_restaurants` function:
```python
from typing import Optional
from sqlalchemy.orm import Query, Session
from models import Restaurant

def get_restaurants(
    session: Session, search: Optional[str] = None, limit: Optional[int] = None
) -> Query:
    filter_args = []
    if search:
        filter_args.append(Restaurant.name.ilike(f"%{search}%"))
    query = session.query(Restaurant).filter(*filter_args)
    if limit:
        query = query.limit(limit)
    return query
``` 
As far as REST API implementation is concerned, we can take parameters from `request.args` structure.
```python
@main.route("/restaurants", methods=["GET"])
def restaurants():
    session = current_app.session
    query = get_restaurants(
        session,
        search=request.args.get("q"),
        limit=request.args.get("limit")
    )
    restaurants = [restaurant_serializer(r) for r in query]
    return jsonify(restaurants)
```
To execute request in insomnia, we use `GET` method and pass parameters in url query string:

![rest_parametrized.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1640180036460/NiZ9thOxm.png)
To parameterize GraphQL connection (list of instances), we must define allowed input arguments. We don't have to define limit parameter because `relay.Connection` class that we use have built in `first`, `last`, `before` and `after` arguments.
```python
class Query(graphene.ObjectType):
    up = graphene.Boolean()
    restaurants = graphene.relay.ConnectionField(
        RestaurantConnection, q=graphene.String()
    )

    def resolve_up(root, info, **kwargs):
        return True

    def resolve_restaurants(root, info, **kwargs):
        query = get_restaurants(
            info.context["session"],
            search=kwargs.get("q"),
            limit=kwargs.get("first")
        )
        return [RestaurantNode(id=r.id, name=r.name) for r in query]

schema = graphene.Schema(query=Query)
```
Here you can see how pass these arguments in GraphQL schema execution:

![graphql_paramertrized.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1640178124421/ItoL4wVA_.png)

## Autogenerated schema
We don't have to define GraphQL schema and REST serialization functions / classes from scratch. For GraphQL we can use `graphene-sqlalchemy` library, while in REST `marshmallow-sqlalchemy` library to build schema upon SQLAlchemy ORM models. It can give a head start and fast iteration but has also a huge drawback. You share the domain model in the public contract (with application clients, like frontend app). So you must consider all pros and cons whether it is worth doing.

## Summary
In this post I showed you how you can start with GraphQL when you are familiar with REST (or in opposite direction). However, I understand that this post does not cover important part of application building, like authentication or authorization. This can be a material for another text. 

Here is a full project implementation: https://github.com/jorzel/service-layer/ because in code listings some parts were omitted for better readability. 

Thanks!
