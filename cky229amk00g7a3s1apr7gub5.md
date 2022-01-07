## GraphQL authentication and authorization

## Introduction
In this second post regarding GraphQL I would like to show how to manage authentication and authorization in GraphQL API. Authentication and authorization are often mixed each other but these concepts are responsible for different processes. The former determine user identity (whether user is logged in or 'recognized' by a system), while the latter refers to whether an authenticated user has access to a given resource. So usually authentication stage precede authorization one. Authentication and authorization could be challenging in GraphQL due to only one exposed HTTP endpoint (e.g. `/graphql`). It would be possible to authenticate user at this endpoint entry, but in that implementation we abandon public access option to some resources. Authorization at this sole endpoint entry is impossible, because we do not know which resources would be queried.

An inspiration for this post has been a [stackoverflow question](https://stackoverflow.com/questions/69126083/set-permissions-on-graphene-relay-node-and-connection-fields/70252461#70252461) that seeks an answer for that topic.

## Application setup
The post is a follow-up for  [my text](https://jorzel.hashnode.dev/graphql-api-and-rest-api-mirror-implementations-in-python) comparing GraphQL and REST example implementations in python. So you could find there a requirements to setup an application.

## Sign Up / Sign In
We start from a simple `User` model that have `email` and hashed `password` properties.  
```python
from sqlalchemy import Column,Integer, String
from sqlalchemy.orm import relationship
from db import Base

class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True, autoincrement=True)
    email = Column(String, unique=True)
    password = Column(String)
    table_bookings = relationship("TableBooking")
```

`User` registration is provided by `SignUp` mutation that takes `email` and `password` arguments and creates `User` record in a database.
```python
# api/graphql.py
import graphene
from service import sign_up

class SignUp(graphene.Mutation):
    class Arguments:
        email = graphene.String(required=True)
        password = graphene.String(required=True)

    user = graphene.Field(UserNode)

    def mutate(self, info, email: str, password: str):
        session = info.context["session"]
        user = sign_up(session, email, password)
        return SignUp(user=user)

class Mutation(graphene.ObjectType):
    sign_up = SignUp.Field()


# service.py
from auth import generate_password_hash

class UserAlreadyExist(Exception):
    pass

def sign_up(session: Session, email: str, password) -> User:
    if session.query(User).filter_by(email=email).first():
        raise UserAlreadyExist()
    user = User(email=email, password=generate_password_hash(password))
    session.add(user)
    session.commit()
    return user


# auth.py
import hashlib

SALT = "STRONg@Salt"

def generate_password_hash(password: str) -> str:
    h = hashlib.md5(f"{password}{SALT}".encode())
    return h.hexdigest()
```
The mutation is executed as a `POST` request at `/graphql` endpoint. As in previous post about GraphQL, I use [insomnia](https://insomnia.rest/) to perform HTTP requests.
![sign_up.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1640694563138/kOoVp6cJF.png)

When `User` instance is created, we need a `SignIn` mutation that generate `User` authentication  JWT `token` if correct credentials are passed. 
```python
# api/graphql.py
import graphene
from service import sign_in

class SignIn(graphene.Mutation):
    class Arguments:
        email = graphene.String(required=True)
        password = graphene.String(required=True)

    token = graphene.String()

    def mutate(self, info, email: str, password: str):
        session = info.context["session"]
        token = sign_in(session, email, password)
        return SignIn(token=token)

class Mutation(graphene.ObjectType):
    sign_in = SignIn.Field()


# service.py
from sqlalchemy.orm import Session
from auth import generate_token, verify_password
from models import User

class UserAuthenticationError(Exception):
    pass

def sign_in(session: Session, email: str, password) -> str:
    user = session.query(User).filter_by(email=email).first()
    if not user:
        raise UserAuthenticationError()
    if not verify_password(user, password):
        raise UserAuthenticationError()
    return generate_token(user)


# auth.py
import hashlib
from itsdangerous import TimedJSONWebSignatureSerializer as Serializer
from models import User

SALT = "STRONg@Salt"
SECRET_KEY = "!SECRET!"
TOKEN_EXPIRES_IN = 3600 * 24 * 30

def generate_password_hash(password: str) -> str:
    h = hashlib.md5(f"{password}{SALT}".encode())
    return h.hexdigest()

def verify_password(user: User, password: str) -> bool:
    return user.password == generate_password_hash(password)

def generate_token(user: User) -> str:
    serializer = Serializer(SECRET_KEY, expires_in=TOKEN_EXPIRES_IN)
    return serializer.dumps({"user_id": user.id}).decode("utf-8")
```
For `SignIn` mutation we pass `email` and `password` and get `token` in payload that can be used in authentication required requests.
![sign_in.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1640695212036/qvufgy-GQ.png)

## Authentication
We have a `token` generated in `SignIn` step, so we can use it in "Bearer Authentication" process. In this kind of authentication, everyone that has valid `token` (bearer) can be recognized as a `User` corresponding to this `token`.
We define a `sign_in_required` decorator that can be used for each GraphQL field resolver. This decorator takes `token` from "Authorization" request header, decode it to get `user_id` and check if `User` corresponding to `user_id` exist. If it completes successfully, we have authenticated `User`. 

```python
# api/graphql.py

import graphene
from api.auth import sign_in_required

class Query(graphene.ObjectType):
    up = graphene.Boolean()
    restaurants = graphene.relay.ConnectionField(
        RestaurantConnection, q=graphene.String()
    )
    me = graphene.Field(UserNode)

    def resolve_up(root, info, **kwargs):
        return True

    @sign_in_required()
    def resolve_restaurants(root, info, **kwargs):
        query = get_restaurants(
            info.context["session"], search=kwargs.get("q"), limit=kwargs.get("first")
        )
        return [RestaurantNode(id=r.id, name=r.name) for r in query]

    @sign_in_required()
    def resolve_me(root, info, **kwargs):
        return kwargs["current_user"]


# api/auth.py
from functools import wraps
from auth import get_user_by_token
from models import User

class UnauthenticatedUser(Exception):
    pass

def sign_in_required():
    def decorator(func):
        @wraps(func)
        def wrapper(root, info, *args, **kwargs):
            kwargs["current_user"] = get_current_user(info.context)
            return func(root, info, *args, **kwargs)
        return wrapper
    return decorator

def get_current_user(context) -> User:
    try:
        token = get_token_from_request(context["request"])
        user = get_user_by_token(context["session"], token)
        if not user:
            raise UnauthenticatedUser("UnauthenticatedUser")
        return user
    except KeyError:
        raise UnauthenticatedUser("UnauthenticatedUser")

def get_token_from_request(request) -> str:
    header = request.headers["Authorization"]
    token = header.replace("Bearer ", "", 1)
    return token

# auth.py
import hashlib
from typing import Optional
from itsdangerous import TimedJSONWebSignatureSerializer as Serializer
from sqlalchemy.orm import Session
from models import User

SECRET_KEY = "!SECRET!"
TOKEN_EXPIRES_IN = 3600 * 24 * 30

def get_user_by_token(session: Session, token: str) -> Optional[User]:
    serializer = Serializer(SECRET_KEY, expires_in=TOKEN_EXPIRES_IN)
    data = serializer.loads(token)
    return session.query(User).get(data["user_id"])
```
`up` field is open access, so we do not have to pass any credentials to query it. On the other hand, `me` field is decorated with `sign_in_required`, so we must pass proper `token` to resolve it.
![me.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1640695457978/ajqtaNpmh.png)
![me_bearer.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1640695462959/YP6VVPAv0.png)
If we hit a field decorated with `sign_in_required` without `token` passed in "Authorization" header, we get an `UnauthenticatedUser` exception.
![endpoint_restricted.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1640695493103/T7w8iNsQ1.png)

## Authorization
We have seen how to authenticate query to restrict access only to signed in users. But, what about a case when we have a `User` that is logged in, but an action that he/she performs is not permitted for him/her. In our example we have two actions:
- booking a restaurant table, that should be allowed if `User` is authenticated,
- cancelling a table booking, that is allowed only for `User` that made this booking before.

We implement two mutatons: `BookRestaurantTable` that has `mutate` method decorated with `sign_in_required` and `CancelTableBooking` with new decorator `authorize_required`used.
This decorator checks `User` is authenticated and also whether `table_booking_gid` (representing global id of an instance) is corresponding to `TableBooking` instance that was created by the authenticated `User`.

```python
class BookRestaurantTable(graphene.Mutation):
    class Arguments:
        restaurant_gid = graphene.ID(required=True)
        persons = graphene.Int(required=True)

    table_booking = graphene.Field(TableBookingNode)

    @sign_in_required()
    def mutate(self, info, restaurant_gid: str, persons: int, **kwargs):
        session = info.context["session"]
        current_user = kwargs["current_user"]
        _, restaurant_id = from_global_id(restaurant_gid)
        table_booking = book_restaurant_table(
            session, restaurant_id, current_user.email, persons
        )
        return BookRestaurantTable(
            table_booking=TableBookingNode(
                id=table_booking.id,
                is_active=table_booking.is_active,
            )
        )


class CancelTableBooking(graphene.Mutation):
    class Arguments:
        table_booking_gid = graphene.ID(required=True)

    table_booking = graphene.Field(TableBookingNode)

    @authorize_required(TableBooking)
    def mutate(self, info, table_booking_gid: str, **kwargs):
        session = info.context["session"]
        table_booking = kwargs["instance"]
        cancel_table_booking(session, table_booking)
        return CancelTableBooking(
            table_booking=TableBookingNode(
                id=table_booking.id,
                is_active=table_booking.is_active,
            )
        )

class Mutation(graphene.ObjectType):
    book_restaurant_table = BookRestaurantTable.Field()
    cancel_table_booking = CancelTableBooking.Field()

# api/auth.py
import re
from functools import wraps
from graphene.relay.node import from_global_id
from auth import authorize
from models import User

def camel_to_snake(name: str) -> str:
    """CamelCase -> camel_case"""
    return re.sub(r"(?<!^)(?=[A-Z])", "_", name).lower()

class UnauthorizedAccess(Exception):
    pass

class InstanceNotExist(Exception):
    pass

def authorize_required(model):
    """
    We assume that global id field name of resource 
    follow convention like:
    model_name: `TableBooking`
    global id field name: `table_booking_gid`
    """
    def decorator(func):
        @wraps(func)
        def wrapper(root, info, *args, **kwargs):
            kwargs["current_user"] = get_current_user(info.context)
            model_name = model.__name__
            gid_field_name = f"{camel_to_snake(model_name)}_gid"
            instance_gid = kwargs[gid_field_name]
            instance_model_name, instance_id = from_global_id(instance_gid)
            if instance_model_name != f"{model_name}Node":
                raise UnauthorizedAccess("UnauthorizedAccess")
            instance = info.context["session"].query(model).get(instance_id)
            if not instance:
                InstanceNotExist()
            kwargs["instance"] = instance
            if not authorize(instance, kwargs["current_user"]):
                raise UnauthorizedAccess("UnauthorizedAccess")
            return func(root, info, *args, **kwargs)
        return wrapper
    return decorator


# auth.py
from functools import singledispatch
from models import TableBooking, User

@singledispatch
def authorize(instance, current_user: User) -> bool:
    raise NotImplementedError

@authorize.register(TableBooking)
def _authorize(instance: TableBooking, current_user: User) -> bool:
    return instance.user_id == current_user.id
```

To execute `BookRestaurantTable` we pass two required arguments: `restaurant_gid` and `persons`. We have to remember about `token` in "Authorization" header. In mutation response we get `TableBooking.id`.
![booking_table_mutation.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1640702766025/2G9eCSp2iU.png)

`CancelTableBooking` takes only `table_booking_gid` that can be taken from `BookRestuarantTable` payload (`TableBooking.id`).
![cancel_booking_mutation.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1640702826325/KRyY8UXJI.png)
If `token` is not corresponding to owner of given `TableBooking`, the action cannot be performed and `UnauthorizedAccess` exception in raised
![unauthorized_access.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1640702831268/24mMer_vT5.png)

## Conclusion
We presented how to perform authentication and authorization steps both for queries and mutations. This implementation is quite generic and can be easily integrated in any python GraphQL project. Full source code you can find here: https://github.com/jorzel/service-layer/tree/auth.

Thanks!
