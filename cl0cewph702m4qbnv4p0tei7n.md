# Value Object persistance strategies


## Overview
Value objects are one of the tactical building blocks for modeling the business domain introduced and popularized by the Domain Driven Design approach.
However, we can exploit value objects even if we do not tackle complex domain problems. Value objects, in contrast to entities, are defined only
by their attributes and do not have any identity. Two value objects are equal when their properties have the same values. Another important trait of a value object is immutability.
Once created cannot be modified. Value object modification always results in creating a new instance of the value object.
Here we have some examples of when we can achieve some gains by value objects implementation:
- distance
- email
- phone
- location
- open hours
- money
- subscription plan

Value objects usually are useful when we encounter the following challenges:
- validation (e.g. encapsulate an email validation)
- various representations (e.g. different representations of a money concept depending on a currency)
- various presentations (e.g. different presentations of a distance concept depending on a country)

However, due to not possessing an identity, value objects can come across some issues with how to persist them.
Here, we suggest four different strategies of persistence value objects in SQLAlchemy ORM (trying to protect value object immutability requirement).
1. [Simple field](#simple-field)
2. [Composite field](#composite-field)
3. [Separated object](#separated-object)
4. [Schemaless document (JSON)](#schemaless-document-json)

## Simple field
In the first approach, we define the `Email` class to encapsulate validation logic, but
persist only a string field column `email_address` in the `shop` table.
```python
# value_objects/email.py
class InvalidEmail(Exception):
    pass

class Email:
    """
    >>>  Email("m.x@test.pl")
    """
    def __init__(self, value: str) -> None:
        if "@" not in value:
            raise InvalidEmail
        self._raw = value
        self._normalized = value.strip().lower()

    def __str__(self) -> str:
        return self._normalized

    __repr__ = __str__

    @property
    def address(self) -> str:
        return self._normalized

    @property
    def domain(self) -> str:
        return self._normalized.split("@")[1]


# models/shop.py
from typing import Optional

from value_objects.email import Email

class Shop:
    def __init__(self):
        self.email_address: Optional[str] = None

    @property
    def email(self) -> Optional[Email]:
        return Email(self.email_address)

    @email.setter
    def email(self, email: Email) -> None:
        self.email_address = email.address


# orm.py
from sqlalchemy import Column, Integer, String, Table
from sqlalchemy.orm import registry

from db import metadata
from models.shop import Shop

mapper_registry = registry()

shop = Table(
    "shop",
    metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("email_address", String),
)

def run_mappers():
    """
    Provides mapping between db tables and domain models.
    """

    mapper_registry.map_imperatively(Shop, shop)

```

## Composite field
In the second approach, we use SQLAlchemy `composite` field to store two fields (`balance_value` as integer and `balance_currency` as the enum) within the `shop` table that corresponds to the `Money` concept.
```python
# value_objects/money.py
import enum

class CurrencyMismatched(Exception):
    pass

class Currency(enum.Enum):
    USD = "USD"
    EUR = "EUR"
    PLN = "PLN"
    CHF = "CHF"

class Money:
    """
    >>>  Money(value=21, currency=Currency.USD)
    """
    def __init__(self, value: int, currency: Currency):
        self._value = value
        self._currency = currency

    def __composite_values__(self):
        return self._value, self._currency

    @property
    def currency(self) -> Currency:
        return self._currency

    @property
    def value(self) -> int:
        return self._value

    def add(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise CurrencyMismatched
        return Money(self.value + other.value, self.currency)


# models/shop.py
from typing import Optional

from value_objects.money import Currency, Money


class Item:
    def __init__(self, price: Money):
        self.price = price


class Shop:
    def __init__(self, balance_currency: Optional[Currency] = Currency.USD):
        if not balance_currency:
            balance_currency = Currency.USD
        self.balance = Money(0, balance_currency)

    def sell_item(self, item: Item) -> None:
        self.balance = self.balance.add(item.price)

# orm.py
from sqlalchemy import Column, Enum, Integer, Table
from sqlalchemy.orm import composite, registry

from db import metadata
from models.shop import Currency, Shop
from value_objects.money import Money

mapper_registry = registry()

shop = Table(
    "shop",
    metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("balance_value", Integer, default=0, nullable=False),
    Column("balance_currency", Enum(Currency), default=Currency.USD, nullable=False),
)


def run_mappers():
    """
    Provides mapping between db tables and domain models.
    """

    mapper_registry.map_imperatively(
        Shop,
        shop,
        properties={
            "balance": composite(Money, shop.c.balance_value, shop.c.balance_currency)
        },
    )

```
## Separated object
The third option stores the value objects in a separated `Location` database table with an `id` column. However, the `id` exist only in the persistence layer.
```python
# value_objects/location.py
class InvalidGeolocation(Exception):
    pass

class Location:
    """
    >>>  Location(city='X', region='Y', longitude=21.11, latitude=20.01)
    """
    def __init__(self, city: str, region: str, longitude: float, latitude: float):
        if longitude < 0 or latitude < 0:
            raise InvalidGeolocation
        self._city = city
        self._region = region
        self._longitude = longitude
        self._latitude = latitude

    @property
    def city(self) -> str:
        return self._city

    @property
    def region(self) -> str:
        return self._region

    @property
    def longitude(self) -> float:
        return self._longitude

    @property
    def latitude(self) -> float:
        return self._latitude

# models/shop.py
from typing import Optional
from value_objects.location import Location

class Shop:
    def __init__(
        self,
        location: Optional[Location] = None,
    ):
        self.location = location


# orm.py
from sqlalchemy import Column, Float, ForeignKey, Integer, String, Table
from sqlalchemy.orm import registry, relationship
from db import metadata
from models.shop import Shop
from value_objects.location import Location

mapper_registry = registry()

location = Table(
    "location",
    metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("city", String),
    Column("region", String),
    Column("longitude", Float),
    Column("latitude", Float),
    UniqueConstraint(
        'city',
        'region',
        'longitude',
        'latitude',
        name='uix_city_region_longitude_latitude'
    )
)

shop = Table(
    "shop",
    metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("location_id", Integer, ForeignKey("location.id")),
)

def run_mappers():
    """
    Provides mapping between db tables and domain models.
    """

    mapper_registry.map_imperatively(
        Location,
        location,
        properties={
            "_city": location.c.city,
            "_region": location.c.region,
            "_latitude": location.c.latitude,
            "_longitude": location.c.longitude,
        },
    )
    mapper_registry.map_imperatively(
        Shop,
        shop,
        properties={
            "location": relationship(Location),
        },
    )

```

## Schemaless document (json)
The last approach persists `OpenHour` concept as a JSON-like document. That data structure is not provided by each database driver but is available in SQLite as a `JSON` field or in PostgreSQL as a `JSONB` field.
```python
# value_objects.open_hours.py
from datetime import datetime
from typing import Any

class OpenHours:
    """
    >>>  OpenHours({'days': [1,2,3,4,5,6], 'hours': [8,9,10,11,12]})
    """
    def __init__(self, config: dict[str, Any]):
        self._config = config

    @property
    def config(self) -> dict[str, Any]:
        return self._config

    @property
    def days(self) -> list[str]:
        return self._config.get("days", [])

    @property
    def hours(self) -> list[str]:
        return self._config.get("hours", [])

    def is_open(self, dt: datetime):
        if dt.isoweekday() not in self.days:
            return False
        if dt.hour not in self.hours:
            return False
        return True

# models/shop.py
from value_objects.open_hours import OpenHours

class Shop:
    def __init__(
        self,
        open_hours_config: Optional[dict] = None,
    ):
        if not open_hours_config:
            open_hours_config = {}
        self.open_hours_config = open_hours_config

    @property
    def open_hours(self) -> OpenHours:
        return OpenHours(self.open_hours_config)

    @open_hours.setter
    def open_hours(self, open_hours: OpenHours) -> None:
        self.open_hours_config = open_hours.config

# orm.py
from sqlalchemy import Column, Integer, Table
from sqlalchemy.dialects.sqlite import JSON
from sqlalchemy.orm import registry

from db import metadata
from models.shop import Shop

mapper_registry = registry()

shop = Table(
    "shop",
    metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("open_hours_config", JSON, nullable=False, default=dict),
)


def run_mappers():
    """
    Provides mapping between db tables and domain models.
    """
    mapper_registry.map_imperatively(
        Shop,
        shop,
    )

```
## Summary
All patterns were implemented using python plain classes and imperative mapping style (to underscore domain and persistence model separation).
Class (value object) properties are read-only to ensure immutability. Because of that, queries must be performed by using tables instead of models (`select(model.__table__).where(...)` instead of `select(model).where(...)`).

This project can also be found in the Github repository: https://github.com/jorzel/value-object-persistance.