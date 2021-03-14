[![Build Status](https://github.com/johnbywater/eventsourcing/actions/workflows/runtests.yaml/badge.svg?branch=9.0)](https://github.com/johnbywater/eventsourcing/tree/9.0)
[![Coverage Status](https://coveralls.io/repos/github/johnbywater/eventsourcing/badge.svg?branch=9.0)](https://coveralls.io/github/johnbywater/eventsourcing?branch=9.0)
[![Documentation Status](https://readthedocs.org/projects/eventsourcing/badge/?version=v9.0.0)](https://eventsourcing.readthedocs.io/en/v9.0.0/)
[![Latest Release](https://badge.fury.io/py/eventsourcing.svg)](https://pypi.org/project/eventsourcing/)


# Event Sourcing in Python

A library for event sourcing in Python.

Please refer to the [documentation](https://eventsourcing.readthedocs.io/) for installation and usage guides.


## Synopsis

The example below uses the library's declarative syntax (new in v9.0)
to define an event sourced aggregate called `World`.
The `World` class uses the `Aggregate` class and the `event`
decorator from the `eventsourcing.domain` module. The `World`
class has a command method `make_it_so()` which triggers an
event called `SomethingHappened` that appends the given value
of `what` to the `history` attribute of an instance of `World`.
It is generally recommended to use an imperative
style when naming command methods, and to name event classes
using past participles.


```python
from eventsourcing.domain import Aggregate, event

class World(Aggregate):
    def __init__(self):
        self.history = []

    @event("SomethingHappened")
    def make_it_so(self, what):
        self.history.append(what)
```

When the `World` aggregate class is called, a created event object is
created and used to construct an instance of `World`. And when the
command method `make_it_so()` is called on an instance of
`World`, a `SomethingHappened` event is triggered with a value of
`what`, which results in the `what` value being appended to the
`history` of the `World` aggregate instance.

```python
# Create a new aggregate.
world = World()

# Execute a command.
world.make_it_so('dinosaurs')
world.make_it_so('trucks')
world.make_it_so('internet')

# View aggregate state.
assert world.history[0] == 'dinosaurs'
```

The resulting events can be collected using the `collect_events()`
method. The `collect_events()` method is used by the `Application`
`save()` method (see below).

```python

# Collect events.
pending_events = world.collect_events()
assert len(pending_events) == 4
assert type(pending_events[0]).__name__ == 'Created'
assert type(pending_events[1]).__name__ == 'SomethingHappened'
assert type(pending_events[2]).__name__ == 'SomethingHappened'
assert type(pending_events[3]).__name__ == 'SomethingHappened'
```

The events can be stored and used to reconstruct the state of the aggregate.

```python

copy = None
for event in pending_events:
    copy = event.mutate(copy)

assert copy.history == world.history
assert copy.id == world.id
assert copy.version == world.version
assert copy.created_on == world.created_on
assert copy.modified_on == world.modified_on
```

This example can be adjusted and extended for any event sourced domain model.

## Installation

Use pip to install the [stable distribution](https://pypi.org/project/eventsourcing/) from
the Python Package Index.

    $ pip install eventsourcing


## Features

**Domain models and applications** — base classes for domain model aggregates
and applications. Suggests how to structure an event-sourced application.

**Flexible event store** — flexible persistence of domain events. Combines
an event mapper and an event recorder in ways that can be easily extended.
Mapper uses a transcoder that can be easily extended to support custom
model object types. Recorders supporting different databases can be easily
substituted and configured with environment variables.

**Application-level encryption and compression** — encrypts and decrypts events inside the
application. This means data will be encrypted in transit across a network ("on the wire")
and at disk level including backups ("at rest"), which is a legal requirement in some
jurisdictions when dealing with personally identifiable information (PII) for example
the EU's GDPR. Compression reduces the size of stored domain events and snapshots, usually
by around 25% to 50% of the original size. Compression reduces the size of data
in the database and decreases transit time across a network.

**Snapshotting** — reduces access-time for aggregates with many domain events.

**Versioning** - allows domain model changes to be introduced after an application
has been deployed. Both domain events and aggregate classes can be versioned.
The recorded state of an older version can be upcast to be compatible with a new
version. Stored events and snapshots are upcast from older versions
to new versions before the event or aggregate object is reconstructed.

**Optimistic concurrency control** — ensures a distributed or horizontally scaled
application doesn't become inconsistent due to concurrent method execution. Leverages
optimistic concurrency controls in adapted database management systems.

**Notifications and projections** — reliable propagation of application
events with pull-based notifications allows the application state to be
projected accurately into replicas, indexes, view models, and other applications.
Supports materialized views and CQRS.

**Event-driven systems** — reliable event processing. Event-driven systems
can be defined independently of particular persistence infrastructure and mode of
running.

**Detailed documentation** — documentation provides general overview, introduction
of concepts, explanation of usage, and detailed descriptions of library classes.

**Worked examples** — includes examples showing how to develop aggregates, applications
and systems.


## Domain model

The examples below explain how the declarative syntax works using
the underlying methods and mechanisms provided by the library's
`Aggregate` base class for event sourced aggregates. Please read the
[documentation](https://eventsourcing.readthedocs.io/) for more information.

The `World` aggregate below has a command method `make_it_so()` which
has an argument `what`. The `make_it_so()` method triggers a domain
event `SomethingHappened` with the value of the `what` arg.
The event class `SomethingHappened` is interpreted as a Python
dataclass, so that the annotation `what: str` is used to define
the class `__init__()` method. The `SomethingHappened` has an
`apply()` method that has a `World` aggregate argument. The
`apply()` method body appends the value of the event's
`what` attribute to the `self.history` of the `World` aggregate.
The method `trigger_event()` is defined on the library's
`Aggregate` base class, and constructs the given event class
and applies the event to the aggregate.


```python

from eventsourcing.domain import Aggregate, AggregateEvent

class World(Aggregate):
    def __init__(self):
        self.history = []

    def make_it_so(self, what: str):
        self.trigger_event(self.SomethingHappened, what=what)

    class SomethingHappened(AggregateEvent):
        what: str

        def apply(self, aggregate: "World"):
            aggregate.history.append(self.what)


world = World()
world.make_it_so('dinosaurs')
assert world.history[0] == 'dinosaurs'
pending_events = world.collect_events()
assert len(pending_events) == 2
assert type(pending_events[0]).__name__ == 'Created'
assert type(pending_events[1]).__name__ == 'SomethingHappened'
```

An alternative way of expressing the same thing is to define a
second method on the aggregate which is decorated with the library's
`@event` decorator. This decorated method will do the work of triggering
and applying an event. The command method can then call the decorated
method rather than triggering an event directly using the `trigger_event()`
method. In this case, the event class is defined without an `apply()`
method, the decorator mentions the event class, and the decorated method
arguments must match the event attributes. When the decorated method is
called, the decorator triggers the event, and the body of the decorated
method is used to apply the event attributes to the aggregate.

```python

from eventsourcing.domain import Aggregate, AggregateEvent, event

class World(Aggregate):
    def __init__(self):
        self.history = []

    def make_it_so(self, what: str):
        self.something_happened(what)

    class SomethingHappened(AggregateEvent):
        what: str

    @event(SomethingHappened)
    def something_happened(self, what: str):
        self.history.append(what)


world = World()
world.make_it_so('dinosaurs')
assert world.history[0] == 'dinosaurs'
pending_events = world.collect_events()
assert len(pending_events) == 2
assert type(pending_events[0]).__name__ == 'Created'
assert type(pending_events[1]).__name__ == 'SomethingHappened'

```

It is sometimes useful to have the event class defined
explicitly, but whenever there is no need to refer to the
event class in other places, the event class can be
automatically generated from the method signature.

A simpler way of expressing the same thing as above is simply
to define an event name in the `@event` decorator using a string.
In this case, the decorator will automatically define an event
class using the decorated method arguments and the given event name.

```python
from eventsourcing.domain import Aggregate, event

class World(Aggregate):
    def __init__(self):
        self.history = []

    def make_it_so(self, what: str):
        self.something_happened(what)

    @event("SomethingHappened")
    def something_happened(self, what: str):
        self.history.append(what)


world = World()
world.make_it_so('dinosaurs')
assert world.history[0] == 'dinosaurs'
pending_events = world.collect_events()
assert len(pending_events) == 2
assert type(pending_events[0]).__name__ == 'Created'
assert type(pending_events[1]).__name__ == 'SomethingHappened'

```

An alternative way of expressing the same thing is to
simply use the `@event` decorator without mentioning an
event class or an event class name. In this case, an event
class name will be constructed from the decorated method name.
The event class name is constructed by splitting the method
name by its underscores, capitalising each part, and then
joining the parts to make the class name `SomethingHappened`.

```python
from eventsourcing.domain import Aggregate, event

class World(Aggregate):
    def __init__(self):
        self.history = []

    def make_it_so(self, what: str):
        self.something_happened(what)

    @event
    def something_happened(self, what: str):
        self.history.append(what)


world = World()
world.make_it_so('dinosaurs')
assert world.history[0] == 'dinosaurs'
pending_events = world.collect_events()
assert len(pending_events) == 2
assert type(pending_events[0]).__name__ == 'Created'
assert type(pending_events[1]).__name__ == 'SomethingHappened'

```

For trivial commands that would simply trigger an event
wilh the given arguments, an alternative way of expressing
the same thing as above is to decorate the command method
with the `@event` decorator. In this case, when the command
method `make_it_so()` is called an event will be triggered,
and the command method body will be used to apply the event
to the aggregate. Because command methods should be named
with imperatives, and events should be named with part
participles, it is recommended to define the name of the event
in the decorator. Commands that need to do some work on the
given arguments before triggering an event will need to use
one of the styles above. Of course, in some cases it may
be more natural to use a past participle as the method name.


```python

from eventsourcing.domain import Aggregate, event

class World(Aggregate):
    def __init__(self):
        self.history = []

    @event("SomethingHappened")
    def make_it_so(self, what):
        self.history.append(what)


world = World()
world.make_it_so('dinosaurs')
assert world.history[0] == 'dinosaurs'
pending_events = world.collect_events()
assert len(pending_events) == 2
assert type(pending_events[0]).__name__ == 'Created'
assert type(pending_events[1]).__name__ == 'SomethingHappened'

```

## Application

Let's define a test that uses the library's event sourced `Application` class
to exercise the `World` aggregate defined above. The `Application` class
brings together a domain model and persistence infrastructure.

```python
from eventsourcing.application import AggregateNotFound, Application
from eventsourcing.system import NotificationLogReader


def test(app: Application):

    # Create a new aggregate.
    world = World()

    # Unsaved aggregate not yet in application repository.
    try:
        app.repository.get(world.id)
    except AggregateNotFound:
        pass

    # Execute aggregate commands.
    world.make_it_so('dinosaurs')
    world.make_it_so('trucks')

    # View current state of aggregate object.
    assert world.history[0] == 'dinosaurs'
    assert world.history[1] == 'trucks'

    # Note version of object at this stage.
    version = world.version

    # Execute another command.
    world.make_it_so('internet')

    # Store pending domain events.
    app.save(world)

    # Aggregate now exists in repository.
    assert app.repository.get(world.id)

    # Show the notification log has four items.
    assert len(app.log['1,10'].items) == 4

    # Replay stored events for aggregate.
    copy = app.repository.get(world.id)

    # View retrieved aggregate.
    assert isinstance(copy, World)
    assert copy.history[0] == 'dinosaurs'
    assert copy.history[1] == 'trucks'
    assert copy.history[2] == 'internet'

    # Get historical state (at version from above).
    old = app.repository.get(world.id, version=version)
    assert old.history[-1] == 'trucks' # internet not happened
    assert len(old.history) == 2

    # Optimistic concurrency control (no branches).
    from eventsourcing.persistence import RecordConflictError

    old.make_it_so('future')
    try:
        app.save(old)
    except RecordConflictError:
        pass
    else:
        raise Exception("Shouldn't get here")

    # Project application event notifications.
    reader = NotificationLogReader(app.log)
    notifications = reader.read(start=1)
    assert len(list(notifications)) == 4

    # - create two more aggregates
    world2 = World()
    app.save(world2)

    world3 = World()
    app.save(world3)

    # - get the new event notifications from the reader
    notifications = reader.read(start=5)
    assert len(list(notifications)) == 2
```

## Development environment

We can run the code in default "development" environment (uses
default "Plain Old Python Object" infrastructure. The example
below runs with no encryption and no compression of the stored
events.

```python

# Construct an application object.
app = Application()

# Run the test.
test(app)

# Records are not encrypted (values are visible in stored data).
def count_visible_values(app):
    num_visible_values = 0
    values = [b'dinosaurs', b'trucks', b'internet']
    reader = NotificationLogReader(app.log)
    for notification in reader.read(start=1):
        for what in values:
            if what in notification.state:
                num_visible_values += 1
                break
    return num_visible_values

assert count_visible_values(app) == 3
```

## SQLite environment

Configure "production" environment using SQLite infrastructure.
The example below uses zlib to compress stored events and AES to
encrypt stored events. An in-memory SQLite database is used, but
it would work the same way if a file path were set as the `SQLITE_DBNAME`.

```python
import os

from eventsourcing.cipher import AESCipher

# Generate a cipher key (keep this safe).
cipher_key = AESCipher.create_key(num_bytes=32)

# Cipher key.
os.environ['CIPHER_KEY'] = cipher_key
# Cipher topic.
os.environ['CIPHER_TOPIC'] = "eventsourcing.cipher:AESCipher"
# Compressor topic.
os.environ['COMPRESSOR_TOPIC'] = "eventsourcing.compressor:ZlibCompressor"

# Use SQLite infrastructure.
os.environ['INFRASTRUCTURE_FACTORY'] = 'eventsourcing.sqlite:Factory'
os.environ['SQLITE_DBNAME'] = ':memory:'  # Or path to a file on disk.
```

Run the code in "production" SQLite environment.

```python
# Construct an application object.
app = Application()

# Run the test.
test(app)

# Records are encrypted (values not visible in stored data).
assert count_visible_values(app) == 0
```

## PostgreSQL environment

Configure "production" environment using PostgresSQL infrastructure.
The example below also uses zlib to compress stored events and AES to
encrypt stored events. It is assumed the database and database
user have already been created, and the database server is running
locally.

```python
import os

from eventsourcing.cipher import AESCipher

# Generate a cipher key (keep this safe).
cipher_key = AESCipher.create_key(num_bytes=32)

# Cipher key.
os.environ['CIPHER_KEY'] = cipher_key
# Cipher topic.
os.environ['CIPHER_TOPIC'] = "eventsourcing.cipher:AESCipher"
# Compressor topic.
os.environ['COMPRESSOR_TOPIC'] = "eventsourcing.compressor:ZlibCompressor"

# Use Postgres infrastructure.
os.environ['INFRASTRUCTURE_FACTORY'] = (
    'eventsourcing.postgres:Factory'
)
os.environ["POSTGRES_DBNAME"] = "eventsourcing"
os.environ["POSTGRES_HOST"] = "127.0.0.1"
os.environ["POSTGRES_USER"] = "eventsourcing"
os.environ["POSTGRES_PASSWORD"] = "eventsourcing"
```

Run the code in "production" PostgreSQL environment.

```python
# Construct an application object.
app = Application()

# Run the test.
test(app)

# Records are encrypted (values not visible in stored data).
assert count_visible_values(app) == 0
```


## Project

This project is [hosted on GitHub](https://github.com/johnbywater/eventsourcing).

Please register questions, requests and [issues on GitHub](https://github.com/johnbywater/eventsourcing/issues), or
post in the project's Slack channel.

There is a [Slack channel](https://join.slack.com/t/eventsourcinginpython/shared_invite/enQtMjczNTc2MzcxNDI0LTJjMmJjYTc3ODQ3M2YwOTMwMDJlODJkMjk3ZmE1MGYyZDM4MjIxODZmYmVkZmJkODRhZDg5N2MwZjk1YzU3NmY)
for this project, which you are [welcome to join](https://join.slack.com/t/eventsourcinginpython/shared_invite/enQtMjczNTc2MzcxNDI0LTJjMmJjYTc3ODQ3M2YwOTMwMDJlODJkMjk3ZmE1MGYyZDM4MjIxODZmYmVkZmJkODRhZDg5N2MwZjk1YzU3NmY).

Please refer to the [documentation](https://eventsourcing.readthedocs.io/) for installation and usage guides.
