## Persistence and domain model separation using SQLAlchemy ORM

## Introduction
You probably have heard about a test pyramid. It is the idea that tan application should have proper balance of automated tests on different layers. There should be a lot of unit tests, significantly less integration tests and a few UI tests (End2End, functional). The reasons for this are maintenence cost and speed of particular test type. Unit tests are usually fast and isolated from the rest of the code (so are easy to setup and maintain). 

![pyramid.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1638049161154/7xCB7w3fJ.png)

That was the theory, but the reality in Python world is rather different. Most of applications exploit ORM (Object Relational Mapper) to setup domain models. So how this applications can be unit tested when domain classes are structurally binded to a database?
The first answer is: Tweak a little bit definition of unit test, to satisfy the need. If domain model tests need a database but are still relatively fast and easy to maintain, we can treat them as unit tests rather than integration tests. But, is it a real solution?
The second answer is: Separate your domain model from persistance model. So, here we go!

## How to separate model?
To show commonly practiced domain and persistance model integration, we take SQLAlchemy library. It is probably the most popular ORM in python community (along with Django ORM). We define `Restaurant` and `Table` models using SQLAlchemy declarative mapping style.
```python
# models.py
from sqlalchemy import Column, MetaData, String, Integer, create_engine
from sqlalchemy.orm import declarative_base, relationship

SQLALCHEMY_DATABASE_URI = "sqlite:///" # your db uri

engine = create_engine(SQLALCHEMY_DATABASE_URI)
metadata = MetaData(bind=engine)
Base = declarative_base(metadata=metadata)

class BookedTableException(Exception):
    pass

class Table(Base):
    id = Column(Integer, primary_key=True, autoincrement=True)
    restaurant_id = Column(Integer, ForeignKey("restaurant.id"))
    max_persons = Column(Integer, default=0)
    is_open = Column(Boolean, default=True)

    def can_book(self, persons: int) -> bool:
        if not self.is_open:
            return False
        if persons > self.max_persons:
            return False
        return True

    def book(self, persons: int) -> None:
        if self.can_book(persons):
            self.is_open = False
        else:
            raise BookedTableException(
                f"{self} cannot be booked, becuase is not open now or is too small"
            )

class Restaurant(Base):
    id = Column(Integer, primary_key=True, autoincrement=True)
    tables = relationship("Table", lazy="dynamic")

    def _get_open_table(self, persons: int) -> Optional[Table]:
        return self.tables.filter(
            Table.max_persons >= persons, 
            Table.is_open.is_(True)
        ).first()

    def has_open_table(self, persons: int) -> bool:
        if self._get_open_table(persons):
            return True
        return False

    def book_table(self, persons: int) -> Optional[Table]:
        table = self._get_open_table(persons)
        if table:
            table.book(persons)
            return table
        raise BookedTableException("No open tables in restaurant")
```
We can see that our integrated model need `Base` object that can be instantiated only if we pass proper databse URI. So to test it we must provide database setup. But, is there any way to avoid it?

### Separated model
```python
# entities.py
from typing import List, Optional

class BookedTableException(Exception):
    pass

class Table:
    def __init__(self, table_id: int, max_persons: int, is_open: bool = True):
        self.id = table_id
        self.max_persons = max_persons
        self.is_open = is_open

    def can_book(self, persons: int) -> bool:
        if not self.is_open:
            return False
        if persons > self.max_persons:
            return False
        return True

    def book(self, persons: int) -> None:
        if self.can_book(persons):
            self.is_open = False
        else:
            raise BookedTableException(
                f"{self} cannot be booked, becuase is not open now or is too small"
            )

class Restaurant:
    def __init__(self, restaurant_id: int, tables: List[Table]):
        self.id = restaurant_id
        self.tables = sorted(tables, key=lambda table: table.max_persons)

    def _get_open_table(self, persons: int) -> Optional[Table]:
        for table in self.tables:
            if table.can_book(persons):
                return table
        return None

    def has_open_table(self, persons: int) -> bool:
        if self._get_open_table(persons):
            return True
        return False

    def book_table(self, persons: int) -> Optional[Table]:
        table = self._get_open_table(persons)
        if table:
            table.book(persons)
            return table
        raise BookedTableException("No open tables in restaurant")

# orm.py
from sqlalchemy import Boolean, Column, ForeignKey, Integer, MetaData, create_engine
from sqlalchemy import Table as sa_Table
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import mapper, relationship
from entities import Restaurant, Table

SQLALCHEMY_DATABASE_URI = "sqlite:///" # your db uri

engine = create_engine(SQLALCHEMY_DATABASE_URI)
metadata = MetaData(bind=engine)
Base = declarative_base(metadata=metadata)

restaurant = sa_Table(
    "restaurant",
    metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
)

table = sa_Table(
    "table",
    metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("restaurant_id", Integer, ForeignKey("restaurant.id")),
    Column("max_persons", Integer),
    Column("is_open", Boolean),
)

def run_mappers():
    """
    Provides mapping between db tables and domain models.
    """
    mapper(
        Restaurant,
        restaurant,
        properties={"tables": relationship(Table, backref="restaurant")},
    )
    mapper(Table, table)

run_mappers() # it should be executed in the app runtime
```
We can see that `models.py` file was divided into `entities.py` file, consisting domain models that are plain python classes and don't know anything about ORM or a database, and `orm.py` file that defines database tables and mapping from a python class to a database table. 

In the end, in `test_entities.py` we test `Restaurant` class method without need of the database setup. Cool? But it's not over yet.
```python
# test_entities.py
import pytest
from entities.restaurant import Restaurant, Table

def test_restaurant_has_open_table_should_pass_if_any_table_in_restaurant_is_open_in_desired_capacity():
    open_table = Table(table_id=1, max_persons=5, is_open=True)
    restaurant = Restaurant(restaurant_id=1, tables=[open_table])

    assert restaurant.has_open_table(3)
```

### What about unit tests at a service layer?
Thanks to the separation we can also unit test the application at a service layer (I have recently written something about service layer abstraction [here](https://jorzel.hashnode.dev/flask-mvt-refactor-to-service-layer)) imitating usage of a database, so the whole application logic can be tested without any integration tests. To make it happen, we need:
- Repository pattern, an abstraction serving access to data storage (e.g. database),
- Unit of Work pattern, an abstraction that groups set of operations into transaction (e.g. database transaction).

We define these abstractions as interfaces and inject them into the service.

```python
# uow.py
from abc import ABC, abstractmethod

class UnitOfWork(ABC):
    is_commited: bool
    is_rollbacked: bool

    @abstractmethod
    def __enter__(self):
        pass

    @abstractmethod
    def __exit__(self, *args):
        pass

    @abstractmethod
    def commit(self):
        pass

    @abstractmethod
    def rollback(self):
        pass

# repositories.py
from abc import ABC, abstractmethod

class RestaurantRepository(ABC):
    @abstractmethod
    def get(self, restaurant_id):
        pass

    @abstractmethod
    def add(self, restaurant):
        pass

# service.py
from uow import UnitOfWork
from repositories import RestaurantRepository

class RestaurantNotExist(Exception):
    pass

class BookingTableService:
    def __init__(
        self,
        restaurant_repository: RestaurantRepository,
        unit_of_work: UnitOfWork,
    ):
        self._restaurant_repository = restaurant_repository
        self._uow = unit_of_work

    def book_table(self, restuarant_id: int, persons: int) -> Table:
        restaurant = self._restaurant_repository.get(restaurant_id)
        if not restaurant:
            raise RestaurantNotExist

        with self._uow:
            restaurant.book_table(persons)
```
For testing needs we can use in-memory imlementation of Repository (```MemoryRestaurantRepository```) and Unit of Work (```MemoryUnitOfWork```), while in production we have Repository (```SQLAlchemyRestaurantRepository```) and Unit of Work (```SQLAlchemyUnitOfWork```)  with a connection to the database.

```python
# uow.py
from sqlalchemy.orm import Session

class MemoryUnitOfWork(UnitOfWork):
    def __init__(self):
        self.is_commited = False
        self.is_rollbacked = False
 
    def __enter__(self):
        self.is_commited = False
        self.is_rollbacked = False
        return self

    def __exit__(self, *args):
        pass

    def commit(self):
        is_commited = True

    def rollback(self):
        is_rollbacked = True

class SQLAlchemyUnitOfWork(UnitOfWork):
    def __init__(self, session: Session):
        self.session = session
        self.is_commited = False
        self.is_rollbacked = False

    def __enter__(self):
        self.is_commited = False
        self.is_rollbacked = False
        return self

    def __exit__(self, *args):
        try:
            self.commit()
        except Exception:
            self.rollback()

    def commit(self):
        self.is_commited = False
        self.session.commit()

    def rollback(self):
        self.is_rollbacked = False
        self.session.rollback()

# repositories.py
from typing import List, Optional
from sqlalchemy.orm import Query, Session
from entities import Restaurant

class MemoryRestaurantRepository(RestaurantRepository):
    def __init__(self):
        self._restaurants = {}

    def get(self, restaurant_id: int) -> Optional[Restaurant]:
        return self._restaurants.get(restaurant_id)

    def add(self, restaurant: Restaurant) -> None:
        self._restaurants[restuarant.id] = restaurant

class SQLAlchemyRestaurantRepository(RestaurantRepository):
    def __init__(self, session: Session):
        self.session = session

    def get(self, restaurant_id: int) -> Optional[Restaurant]:
        return self.session.query(Restuarant).get(restaurant_id)

    def add(self, restaurant: Restaurant) -> None:
        self.session.add(restaurant)
        self.session.flush()

# test_service.py
from typing import List, Optional
import pytest
from service import BookingTableService
from repositories import MemoryRestaurantRepository
from uow import MemoryUnitOfWork

@pytest.fixture
def restaurant_factory():
    def _restaurant_factory(
        restaurant_id: int,
        tables: Optional[List[Table]] = None,
        repository: RestaurantRepository = MemoryRestaurantRepository(),
    ):
        if not tables:
            tables = []
        restaurant = Restaurant(restaurant_id, tables)
        repository.add(restaurant)
        return restaurant
    yield _restaurant_factory

def test_booking_service_book_table_should_pass_when_table_in_restaurant_is_available(
    restaurant_factory
):
    repository = MemoryRestaurantRepository()
    uow = MemoryUnitOfWork()
    booking_service = BookingTableService(repository, uow)
    table = Table(table_id=1, max_persons=5, is_open=True)
    restaurant = restaurant_factory(
        restaurant_id=1, tables=[table], repository=repository
    )

    booking_service.book_table(restaurant_id=restaurant.id, persons=3)

    assert table.is_open is False
    assert uow.is_committed is True
```

## Is the separation always a good idea?
We have seen that there is a need of some additional patterns (Dependency Injection, Repository, Unit of Work) and boilerplate code in this solution. So, it must be carefully considered whether given project need it. Here are some tips when we should go for it, becuase it brings some value and when we should avoid it, because it would be overkill or unnecessary complication.

For arguments:
- unit testing without database setup.
- infrastructure resposibility separated from domain. You can focus on business rules when your domain model is rich,
- your model does not have direct mapping between persistance and domain, e.g. eventsourcing (you persist stream of events that are projected into stateful aggregate). 

Against arguments:
- connascene - code that is changed for the same reason should be in the same place. Assuming, we have direct mapping between domain model classes and tables in database, the change in domain model, trigger the change in persistance model and database table (more about connascene: https://connascence.io/),
- easier and more popular SQLAlchemy declarative mapping API,
- your project is a CRUD application. It doesn't follow classical test pyramid. You probably can test whole application through API (functional / integration tests), because there is no business logic,
- performance issues, e.g. for filtering `tables` in integrated model we can execute indexed database query, while in domain model we have to iterate through the whole collection (for large collections it could be a problem).

That's all. 

If you find this subject interesting, here is full implementation: https://github.com/jorzel/opentable/.
I also recommend great book and code repository that was inspiration for writing this post: https://github.com/cosmicpython/book.
