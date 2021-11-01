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