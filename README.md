# Overview
Some examples of good object oriented coding practices (e.g. SOLID)

# Table of Contents
  1. [SOLID](#solid)
     1. [**S**ingle Responsibility Principle (SRP)](#single-responsibility-principle-srp)
     2. [**O**pen/Closed Principle (OCP)](#openclosed-principle-ocp)
     3. [**L**iskov Substitution Principle (LSP)](#liskov-substitution-principle-lsp)
     4. [**I**nterface Segregation Principle (ISP)](#interface-segregation-principle-isp)
     5. [**D**ependency Inversion Principle (DIP)](#dependency-inversion-principle-dip)

## SOLID
### Single Responsibility Principle (SRP)
### Open/Closed Principle (OCP)
This principle insists that implementing a new feature should add new code without modifing existing one.
We can see now that implementing new types of `Period` force us to modify `create_period_by_name` function.

```python
from abc import ABC, abstractmethod
from datetime import datetime, timedelta

class Period(ABC):
    name: str

    @property
    @abstractmethod
    def start(self) -> datetime:
        pass

    @property
    @abstractmethod
    def end(self) -> datetime:
        pass


class Today(Period):
    name = 'Today'

    def __init__(self):
        self._dt = datetime.now()

    @property
    def start(self) -> datetime:
        return datetime(self._dt.year, self._dt.month, self._dt.day, 0, 0, 0)

    @property
    def end(self) -> datetime:
        return datetime(self._dt.year, self._dt.month, self._dt.day, 23, 59, 59)


class RecentWeek(Period):
    name = 'RecentWeek'

    def __init__(self):
        self._now = datetime.now()
        self._ago = self._now - timedelta(days=7)

    @property
    def start(self) -> datetime:
        return datetime(self._ago.year, self._ago.month, self._ago.day, 0, 0, 0)

    @property
    def end(self) -> datetime:
        return datetime(self._now.year, self._now.month, self._now.day, 23, 59, 59)


def create_period_by_name(name: str):
    if name == Today.name:
        return Today()
    elif name == Yesterday.name:
        return Yesterday()
    else:
        raise NotImplementedError
```
Transformation of `create_period_by_name` function in the way that introduction of new `Period` types
is possible without any modifications of `create_period_by_name` function.

```python
def create_period_by_name(name: str):
    for period_cls in Period.__subclasses__():
        if period_cls.name == name:
            return period_cls()
    raise NotImplementedError
```

### Liskov Substitution Principle (LSP)
### Interface Segregation Principle (ISP)
### Dependency Inversion Principle (DIP)
We have a class `BookingTableService`, dependent on class `SQLAlchemyRestaurantRepo`. 

```python
from typing import Optional

from sqlalchemy.orm import Session


class Restaurant:
    def __init__(self, restaurant_id: str):
        self.id = restaurant_id
    
    def book_table(self):
        # some implementation


class SQLAlchemyRestaurantRepo:
    def ___init__(self, session: Session):
        self._session = session

    def get(self, restaurant_id: str) -> Optional[Restaurant]:
        return self._session.query(Restaurant).get(restaurant_id)


class BookingTableService:
    def __init__(self, session: Session):
        self._repo = SQLAlchemyRestaurantRepo(session) 

    def book_table(restaurant_id: str) -> None:
        restaurant = self._repo.get(restaurant_id)
        restaurant.book_table()


r = Restaurant(restaurant_id=1)
session = Session()
service = BookingTableService(session)
service.book_table(r.id)
```

We can modify the code, and try to decrease coupling between `BookingTableService` and `SQLAlchemyRestaurntRepo`
by introducing an abstraction (interface) `RestaurantRepo` that class `BookingTableService` and class `SQLAlchemyRestaurantRepo` are dependent on.

```python
from abc import ABC, abstractmethod
from typing import Optional

from sqlalchemy.orm import Session


class Restaurant:
    def __init__(self, restaurant_id: str):
        self.id = restaurant_id

    def book_table(self):
        # some implementation


class RestaurantRepo(ABC):
    @abstractmethod
    def get(self, restaurant_id: str) -> Optional[Restaurant]:
        pass


class SQLAlchemyRestaurantRepo(RestuarantRepo):
    def ___init__(self, session: Session):
        self._session = session

    def get(self, restaurant_id: str) -> Optional[Restaurant]:
        return self._session.query(Restaurant).get(restaurant_id)


class BookingTableService:
    def __init__(self, restaurant_repo: RestaurantRepo):
        self._repo = resturant_repo

    def book_table(restaurant_id: str) -> None:
        restaurant = self._repo.get(restaurant_id)
        restaurant.book_table()


r = Restaurant(restaurant_id=1)
session = Session()
service = BookingTableService(SQLAlchemyRestaurantRepo(session))
service.book_table(r.id)
```