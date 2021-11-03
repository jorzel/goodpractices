# Overview
Some examples of good object oriented coding practices (e.g. SOLID)

# Table of Contents
1. [SOLID](#solid)
     1. [**S**ingle Responsibility Principle (SRP)](#single-responsibility-principle-srp)
     2. [**O**pen/Closed Principle (OCP)](#openclosed-principle-ocp)
     3. [**L**iskov Substitution Principle (LSP)](#liskov-substitution-principle-lsp)
     4. [**I**nterface Segregation Principle (ISP)](#interface-segregation-principle-isp)
     5. [**D**ependency Inversion Principle (DIP)](#dependency-inversion-principle-dip)
2. [Coupling](#coupling)
    1. [Private method](#private-method)
    2. [Command delegation to executor (instance without Dependency Injection)](#command-delegation-to-executor-instance-without-dependency-injection)
    3. [Command delegation to executor (instance with Dependency Injection)](#command-delegation-to-executor-instance-with-dependency-injection)
    4. [Command delegation to executor (interface with Dependency Injection)](#command-delegation-to-executor-interface-with-dependency-injection)
    5. [Event dispatch (with Dependency Injection)](#event-dispatch-with-dependency-injection)

## SOLID
### Single Responsibility Principle (SRP)
This principle states that class/function should do one thing and have only one reason to change.
In the following example, method `book_table` is responsible for domain logic (booking table) and
also some infrastructure code (sending notification).
```python
from dataclasses import dataclass
from typing import List, Optional


@dataclass
class User:
    phone_number: str

class Table:
    def __init__(self):
        self.is_booked = False

    def book(self) -> None:
        self.is_booked = True


class SmsSender():
    def send(self, user: User, text: str) -> None:
        # some implementation


class Restaurant:
    def __init__(self, restaurant_id: str, tables: List[Table]):
        self.id = restaurant_id
        self.tables = tables

    def book_table(self, user: User) -> None:
        table = self._find_open_table()
        if table:
            table.book()
            self._send_notification(user.phone_number)
        else:
            # exception or logging

    def _find_open_table(self) -> Optional[Table]:
        for table in self.tables:
            if not table.is_booked:
                return table
        return None

    def _send_notification(self, user: User) -> None:
        sender = SmsSender()
        sender.send(user, "Send booked table notification")  
```
We can improve this code by moving out sending notification responsibility from `Restuarant` class.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import List, Optional


@dataclass
class User:
    phone_number: str


class Table:
    def __init__(self):
        self.is_booked = False

    def book(self) -> None:
        self.is_booked = True


class NotificationSender(ABC):
    @abstractmethod
    def send(self, user: User, text: str) -> None:
        pass


class Restaurant:
    def __init__(self, restaurant_id: str, tables: List: Table):
        self.id = restaurant_id
        self.tables = tables

    def book_table(self) -> None:
        table = self._find_open_table()
        if table:
            table.book()
        else:
            # exception or logging

    def _find_open_table(self) -> Optional[Table]:
        for table in self.tables:
            if not table.is_booked:
                return table
        return None


class RestaurantRepo(ABC):
    @abstractmethod
    def get(self, restaurant_id: str) -> Optional[Restaurant]:
        pass


class BookingTableService:
    def __init__(self, notification_sender: NotificationSender, restaurant_repo: RestaurantRepo):
        self._restaurant_repo = restaurant_repo
        self._notification_sender = notification_sender

    def book_table(self, restaurant_id: str, user: User) -> None:
        restaurant = self._restuarant_repo.get(restuarant_id)
        restuarant.book_table()
        self._notification_sender.send(user, "Send booked table notification")

```

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
We can change function `create_period_by_name` implementation in the way that introduction of new `Period` types
is possible without any modifications of `create_period_by_name` function.

```python
def create_period_by_name(name: str):
    for period_cls in Period.__subclasses__():
        if period_cls.name == name:
            return period_cls()
    raise NotImplementedError
```

### Liskov Substitution Principle (LSP)
This principle claims that subtypes should not change interface of methods inherited from parent type.
Thanks to it, parent type can be replaced by its subtype without breaking the application. In code below
`SQLAlchemyRestaurantRepo` breaks inherited interface by not implementing `add` method.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import datetime
from typing import Optional

from sqlalchemy.orm import Session


@dataclass(frozen=True)
class Event:
    name: str
    originator_id: str
    timestamp: datetime = datetime.now


class Restaurant:
    def __init__(self, restaurant_id: str, events: List[Event]):
        self.id = restaurant_id
        self.status: str
        for event in events:
            self.apply(event)
        self._changes = []

    @property
    def changes(self) -> List[Event]:
        return self._changes[:]

    def _register_change(self, event: Event) -> None:
        self._changes.append(event)

    def apply(self, event: Event) -> None:
        if event.name == 'booked_table':
            self._book_table(event)

    def book_table(self) -> None:
        self._register_change(Event(name='booked_table', originator_id=self.id))

    def _book_table(self, event: Event) -> None:
        # some implementation


class RestaurantRepo(ABC):
    @abstractmethod
    def get(self, restaurant_id: str) -> Optional[Restaurant]:
        pass

    @abstractmethod
    def add(self, restaurant: Restaurant) -> None:
        pass

    @abstractmethod
    def save(self, restaurant: Restaurant) -> None:
        pass


class SQLAlchemyRestaurantRepo(RestaurantRepo):
    def __init__(self, session: Session):
        self._session = session

    def get(self, restaurant_id: str) -> Optional[Restaurant]:
        events = self._session.query(Event).filter_by(originator_id=restaurant_id)
        if events:
            return Restaurant(restaurant_id, events)
        return None

    def add(self, restaurant: Restaurant) -> None:
        raise NotImplementedError

    def save(self, restaurant: Restaurant) -> None:
        for event in restaurant.changes:
            self._session..add(event)
```

We can fix LSP violation by using more compatible interface that is inherited by `SQLAlchemyRestaurantRepo`.

```python
class EventStoreRestaurantRepo(ABC):
    @abstractmethod
    def get(self, restaurant_id: str) -> Optional[Restaurant]:
        pass

    @abstractmethod
    def save(self, restaurant: Restaurant) -> None:
        pass


class SQLAlchemyRestaurantRepo(EventStoreRestaurantRepo):
    def __init__(self, session: Session):
        self._session = session

    def get(self, restaurant_id: str) -> Optional[Restaurant]:
        events = self._session.query(Event).filter_by(originator_id=restaurant_id)
        if events:
            return Restaurant(restaurant_id, events)
        return None

    def save(self, restaurant: Restaurant) -> None:
        for event in restaurant.changes:
            self._session.add(event)
```

### Interface Segregation Principle (ISP)
This principles states that interface of a class should be coherent and not outgrown.
Following implementation violetes ISP, because `BookingTableService` depends on a code that 
is not used (`NotificationSender.send_email`).
```python
class User:
    phone_number: str
    email: str


class NotificationSender:
    def send_sms(self, user: User, text: str) -> None:
        # some implementation

    def send_email(self, user: User, text: str) -> None:
        # some implmenetation        


class BookingTableService:
    def __init__(self, notification_sender: NotificationSender):
        self._notification_sender = notification_sender

    def book_table(self, restaurant_id: str, user: User) -> None:
        # some logic
        self._notification_sender.send_sms(user, "Send booked table notification")


BookingTableService(NotificationSender()).book_table(
    1, User(phone_number='12223332', email='test@test.pl')
)
```
We can implement `NotificationSender` as separated classes and avoid inheriting not necessary code.
```python
from abc import ABC, abstractmethod


class User:
    phone_number: str
    email: str


class NotificationSender(ABC):
    @abstractmethod
    def send(self, user: User, text: str) -> None:
        pass


class SmsSender(NotificationSender):
    def send(self, user: User, text: str) -> None:
        # some implementation


class EmailSender(NotificationSender):
    def send(self, user: User, text: str) -> None:
        # some implementation


class BookingTableService:
    def __init__(self, notification_sender: NotificationSender):
        self._notification_sender = notification_sender

    def book_table(self, restaurant_id: str, user: User) -> None:
        # some logic
        self._notification_sender.send(user, "Send booked table notification")


BookingTableService(SmsSender()).book_table(
    1, User(phone_number='12223332', email='test@test.pl')
)
```

### Dependency Inversion Principle (DIP)
This principle states that classes should be loosely coupled and high-level modules
should not import anything from low-level modules.
In our example, we have a class `BookingTableService` dependent on class `SQLAlchemyRestaurantRepo`. 

```python
from typing import Optional

from sqlalchemy.orm import Session


class Restaurant:
    def __init__(self, restaurant_id: str):
        self.id = restaurant_id
    
    def book_table(self) -> None:
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

    def book_table(self) -> None:
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

## Coupling
Coupling is a degree of dependence between classes/modules/functions.
The table below defines coupling level by amount of knowledge about a remote/delegated action (notification in our example) and the action executor.
|                                                                        | How action is done? | Where is executor instantiated? | What type executor is? | What action? | Coupling level |
|------------------------------------------------------------------------|---------------------|---------------------------------|------------------------|--------------|----------------|
| Private method                                                         | Yes                 | Yes                             | Yes                    | Yes          | Very Strong    |
| Command Delegation to executor (instance without Dependency Injection) | No                  | Yes                             | Yes                    | Yes          | Strong         |
| Command Delegation to executor   (instance with Dependency Injection)  | No                  | No                              | Yes                    | Yes          | Medium         |
| Command Delegation to executor (interface with Dependency Injection)   | No                  | No                              | No                     | Yes          | Loose          |
| Event dispatch (with Dependency Injection)                             | No                  | No                              | No                     | No           | Very Loose     |

### Private method
```python
class BookingTableService:
    def book_table(restaurant_id: str) -> None:
        # some implemenation
        self._send_notification(restaurant_id)

    def _send_notification(self, restaurant_id: str) -> None
        logging.info(f'Table booked in {restaurant_id=}')


BookingTableSerice().book_table(1)
```
### Command delegation to executor (instance without Dependency Injection)
```python
class NotificationSender:
    def send(self, **kwargs)-> None:
        # some implementation


class BookingTableService:
    def __init__(self):
        self._notification_sender = NotificationSender()

    def book_table(restaurant_id: str) -> None:
        # some implemenation
        self._notification_sender.send(recruitment_id=recruitment_id)


BookingTableSerice().book_table(1)
```
### Command Delegation to executor (instance with Dependency Injection)
```python
class NotificationSender:
    def send(self, **kwargs) -> None:
        # some implementation


class BookingTableService:
    def __init__(self, notification_sender: NotificationSender):
        self._notification_sender = notification_sender

    def book_table(restaurant_id: str) -> None:
        # some implemenation
        self._notification_sender.send(recruitment_id=recruitment_id)


BookingTableSerice(notification_sender=NotificationSender()).book_table(1)
```
### Command Delegation to executor (interface with Dependency Injection)
```python
from abc import ABC, abstractmethod

class NotificationSender(ABC):
    @abstractmethod
    def send(self, **kwargs) -> None:
        pass


class SmsSender(NotificationSender):
    def send(self, **kwargs) -> None:
        # some implementation


class BookingTableService:
    def __init__(self, notification_sender: NotificationSender):
        self._notification_sender = notification_sender

    def book_table(restaurant_id: str) -> None:
        # some implemenation
        self._notification_sender.send(recruitment_id=recruitment_id)


BookingTableSerice(notification_sender=SmsSender()).book_table(1)
```
### Event Dispatch (with Dependency Injection)
```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import datetime
from typing import List


@dataclass(frozen=True)
class Event:
    name: str
    originatior_id: str
    timestamp: datetime = datetime.now()


class EventHandler:
    def handle(self, events: List[Events]) -> None:
        for event in events:
            if event.name == 'booked_table':
                # some implementation of sending notification


class EventDispatcher(ABC):
    @abstractmethod
    def publish(self, events: List[Event]) -> None:
        pass


class FakeEventDispatcher(EventDispatcher):
    def publish(self, events: List[Event]) -> None:
        handler = EventHandler()
        handler.handle(events)


class BookingTableService:
    def __init__(self, event_dispatcher: EventDispatcher):
        self._event_dispatcher = event_dispatcher

    def book_table(restaurant_id: str) -> None:
        # some implemenation
        events = [Event(name="booked_table", originator_id=restaurant_id)]
        self._event_dispatcher.publish(events)


BookingTableSerice(event_dispatcher=FakeEventDispatcher()).book_table(1)
```