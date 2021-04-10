==================================================
:mod:`~eventsourcing.application` --- Applications
==================================================

This module helps with developing event-sourced applications.

An event-sourced **application** object has command and query
methods used by clients to interact with its domain model.
An application object has an event-sourced **repository** used to obtain already
existing event-sourced aggregates. It also has a **notification log**
that is used to propagate the state of the application as a sequence
of domain event notifications.


Domain-driven design
====================

The book *Domain-Driven Design* describes a "layered architecture" with four layers:
interface, application, domain, and infrastructure. The application layer depends on
the domain and infrastructure layers. The interface layer depends on the application
layer.

Generally speaking, the application layer implements commands which change the
state of the application, and queries which present the state of the application.
The commands and queries ("application services") are called from the interface layer.
By keeping the application and domain logic in the application and domain layers,
different interfaces can be developed for different technologies without duplicating
application and domain logic.

The discussion below continues these ideas, by combining event-sourced aggregates
and persistence objects in an application object that implements "application services"
as object methods.

.. _Application objects:

Application objects
===================

An event-sourced application object combines a domain model with
a cohesive mechanism for storing and retrieving domain events.

The library's :class:`~eventsourcing.application.Application` object class
brings together objects from the :doc:`domain </topics/domain>` and
:doc:`persistence </topics/persistence>` modules. It can be subclassed
to develop event-sourced applications. The general idea is to name
your application object class after the domain supported by its domain model,
and then define command and query methods that allow interfaces to create, read,
update and delete your domain model aggregates. Domain model aggregates are discussed
in the :doc:`domain module documentation </topics/domain>`. The "ubiquitous language"
of your project should guide the names of the application's command and query methods,
along with those of its :ref:`domain model aggregates <Aggregates>`.

The :class:`~eventsourcing.application.Application` class defines an object method
:func:`~eventsourcing.application.Application.save` which can be
used to update the recorded state of one or many
:ref:`domain model aggregates <Aggregates>`. The
:func:`~eventsourcing.application.Application.save` method functions by using
the aggregate's :func:`~eventsourcing.domain.Aggregate.collect_events` method to collect
pending domain events; the pending domain events are stored by calling the
:func:`~eventsourcing.persistence.EventStore.put` method of application's
:ref:`event store <Store>`.

The :class:`~eventsourcing.application.Application` class defines an
object attribute ``repository`` which holds an :ref:`event-sourced repository <Repository>`.
The repository's :func:`~eventsourcing.application.Repository.get` method can be used by
your application's command and query methods to obtain already existing aggregates.

The :class:`~eventsourcing.application.Application` class defines an
object attribute ``log`` which holds a :ref:`local notification log <Notification log>`.
The notification log can be used to propagate the state of an application as a sequence of
domain event notifications.

The :class:`~eventsourcing.application.Application` class defines an object method
:func:`~eventsourcing.application.Application.take_snapshot` which can
be used for :ref:`snapshotting <Snapshotting>` existing aggregates. Snapshotting
isn't necessary, but can help to reduce the time it takes to access aggregates with
lots of domain events.


Basic example
=============

In the example below, the ``Worlds`` application extends the library's
application object base class. The ``World`` aggregate is defined and discussed
as the :ref:`basic example <Aggregate basic example>` in the domain module documentation.

The ``Worlds`` application's ``create_world()`` method is a command method that creates
and saves new ``World`` aggregates, returning a new ``world_id`` that can be
used to identify the aggregate on subsequence method calls. It saves the new
aggregate by calling the base class ``save()`` method.

The ``Worlds`` application's ``make_it_so()`` method is a command method that obtains an
existing ``World`` aggregate from the repository, then calls the aggregate's
command method ``make_it_so()``, and then saves the aggregate by calling the
application's ``save()`` method.

The ``Worlds`` application's ``get_world_history()`` method is a query method that
presents the current history of an existing aggregate.

.. code:: python

    from typing import List
    from uuid import UUID

    from eventsourcing.application import Application


    class Worlds(Application):
        def create_world(self) -> UUID:
            world = World.create()
            self.save(world)
            return world.id

        def make_it_so(self, world_id: UUID, what: str):
            world = self.repository.get(world_id)
            world.make_it_so(what)
            self.save(world)

        def get_world_history(self, world_id: UUID) -> List[str]:
            world = self.repository.get(world_id)
            return list(world.history)


..
    from dataclasses import dataclass
    from uuid import uuid4

    from eventsourcing.domain import Aggregate, AggregateEvent


    class World(Aggregate):
        def __init__(self):
            self.history = []

        @classmethod
        def create(cls):
            return cls._create(
                event_class=cls.Created,
                id=uuid4(),
            )

        def make_it_so(self, what):
            self.trigger_event(World.SomethingHappened, what=what)

        class SomethingHappened(AggregateEvent):
            what: str

            def apply(self, world):
                world.history.append(self.what)


In the example below, an instance of the ``Worlds`` application is constructed.
A new ``World`` aggregate is created by calling the ``create_world()`` method.
Three items are added to its history: "dinosaurs", "trucks", and "internet" by
calling the ``make_it_so()`` application command with the ``world_id`` aggregate
ID. The history of the aggregate is obtained when the ``get_world_history()``
method is called.


.. code:: python

    application = Worlds()

    world_id = application.create_world()

    application.make_it_so(world_id, "dinosaurs")
    application.make_it_so(world_id, "trucks")
    application.make_it_so(world_id, "internet")

    history = application.get_world_history(world_id)
    assert history[0] == "dinosaurs"
    assert history[1] == "trucks"
    assert history[2] == "internet"

By default, the application object uses the “Plain Old Python Object” infrastructure
which has stored domain events in memory only. To store the domain events in a real
database, you will need to :ref:`configure persistence<Persistence>`.

.. _Repository:

Repository
==========

A repository is used to get the already existing aggregates of the application's domain model.

The application object's ``repository`` attribute has an instance of the
library's :class:`~eventsourcing.application.Repository` class.

The repository's :func:`~eventsourcing.application.Repository.get` method is used to
obtain already existing aggregates. It uses the event store's
:func:`~eventsourcing.persistence.EventStore.get` method to retrieve
the already existing :ref:`domain event objects <Events>` of the requested
aggregate, and the :func:`~eventsourcing.domain.AggregateEvent.mutate`
methods of the :ref:`domain event objects <Events>` to reconstruct the state
of the requested aggregate. The repository's
:func:`~eventsourcing.application.Repository.get` method accepts two
arguments: ``aggregate_id`` and ``version``:

The ``aggregate_id`` argument is required, and should be the ID of an already existing
aggregate. If the aggregate is not found, the exception
:class:`~eventsourcing.application.AggregateNotFound` will be raised.

The ``version`` argument is optional, and represents the required version of the aggregate.
If the requested version is greater than the highest available version of the aggregate, the
highest available version of the aggregate will be returned.


.. code:: python

    world_latest = application.repository.get(world_id)

    assert world_latest.version == 4
    assert len(world_latest.history) == 3


    world_v1 = application.repository.get(world_id, version=1)

    assert world_v1.version == 1
    assert len(world_v1.history) == 0

    world_v2 = application.repository.get(world_id, version=2)

    assert world_v2.version == 2
    assert len(world_v2.history) == 1
    assert world_v2.history[-1] == "dinosaurs"

    world_v3 = application.repository.get(world_id, version=3)

    assert world_v3.version == 3
    assert len(world_v3.history) == 2
    assert world_v3.history[-1] == "trucks"

    world_v4 = application.repository.get(world_id, version=4)

    assert world_v4.version == 4
    assert len(world_v4.history) == 3
    assert world_v4.history[-1] == "internet"

    world_v5 = application.repository.get(world_id, version=5)

    assert world_v5.version == 4  # There is no version 5.
    assert len(world_v5.history) == 3
    assert world_v5.history[-1] == "internet"


.. _Notification Log:

Notification log
================

A notification log can be used to propagate the state of an application as a
sequence of domain event notifications.

The application object's ``log`` attribute has an instance of the library's
:class:`~eventsourcing.application.LocalNotificationLog` class. The notification
log presents linked sections of
:ref:`notification objects <Notification objects>`.
The sections are instances of the library's :class:`~eventsourcing.application.Section` class.

Each event notification has an ``id`` that has the unique integer ID of
the event notification. The event notifications are ordered by their IDs,
with later event notifications having higher values than earlier ones.

A notification log section is identified by a section ID string that comprises
two integers separated by a comma, for example ``"1,10"``. The first integer
specifies the notification ID of the first event notification included in the
section. The second integer specifies the notification ID of the second event
notification included in the section. Sections are requested from the notification
using the Python square bracket syntax, for example ``application.log["1,10"]``.

The notification log will return a section that has no more than the requested
number of event notifications. Sometimes there will be less event notifications
in the recorded sequence of event notifications than are needed to fill the
section, in which case less than the number of event notifications will be included
in the returned section. On the other hand, there may be gaps in the recorded
sequence of event notifications, in which case the last event notification
included in the section may have a notification ID that is greater than that
which was specified in the requested section ID.

A notification log section has an attribute ``section_id`` that has the section
ID. The section ID value will represent the event notification ID of the first
and the last event notification included in the section. If there are no event
notifications, the section ID will be ``None``.

A notification log section has an attribute ``items`` that has the list of
:ref:`notification objects <Notification objects>` included in the section.

A notification log section has an attribute ``next_id`` that has the section ID
of the next section in the notification log. If the notification log section has
less event notifications that were requested, the ``next_id`` value will be ``None``.

In the example above, there are four domain events in the domain model, and so there
are four notifications in the notification log.

.. code:: python

    from eventsourcing.persistence import Notification

    section = application.log["1,10"]

    assert len(section.items) == 4
    assert section.id == "1,4"
    assert section.next_id is None

    assert isinstance(section.items[0], Notification)
    assert section.items[0].id == 1
    assert section.items[1].id == 2
    assert section.items[2].id == 3
    assert section.items[3].id == 4

    assert section.items[0].originator_id == world_id
    assert section.items[1].originator_id == world_id
    assert section.items[2].originator_id == world_id
    assert section.items[3].originator_id == world_id

    assert section.items[0].originator_version == 1
    assert section.items[1].originator_version == 2
    assert section.items[2].originator_version == 3
    assert section.items[3].originator_version == 4

    assert "World.Created" in section.items[0].topic
    assert "World.SomethingHappened" in section.items[1].topic
    assert "World.SomethingHappened" in section.items[2].topic
    assert "World.SomethingHappened" in section.items[3].topic

    assert b"dinosaurs" in section.items[1].state
    assert b"trucks" in section.items[2].state
    assert b"internet" in section.items[3].state

A domain event can be reconstructed from an event notification by calling the
application's mapper method :func:`~eventsourcing.persistence.Mapper.to_domain_event`.
If the application is configured to encrypt stored events, the event notification
will also be encrypted, but the mapper will decrypt the event notification.

.. code:: python

    domain_event = application.mapper.to_domain_event(section.items[0])
    assert isinstance(domain_event, World.Created)
    assert domain_event.originator_id == world_id

    domain_event = application.mapper.to_domain_event(section.items[3])
    assert isinstance(domain_event, World.SomethingHappened)
    assert domain_event.originator_id == world_id
    assert domain_event.what == "internet"

.. _Snapshotting:

Snapshotting
============

If the reconstruction of an aggregate depends on obtaining and replaying
a relatively large number of domain event objects, it can take a relatively
long time to reconstruct the aggregate. Snapshotting aggregates can help to
reduce access time of aggregates with lots of domain events.

The application method :func:`~eventsourcing.application.Application.take_snapshot`
can be used to create a snapshot of the state of an aggregate. The ID of an aggregate
to be snapshotted must be passed when calling this method. By passing in the ID
(and optional version number), rather than an actual aggregate object, the risk of
snapshotting a somehow "corrupted" aggregate object that does not represent the
actually recorded state of the aggregate is avoided.

To enable the snapshotting functionality, the environment variable
``IS_SNAPSHOTTING_ENABLED`` must be set to a valid "true"  value. The
function :func:`~distutils.utils.strtobool` from the Python :mod:`distutils.utils`
module is used to interpret the value of this environment variable, so that strings
``"y"``, ``"yes"``, ``"t"``, ``"true"``, ``"on"`` and ``"1"`` are considered to
be "true" values, and ``"n"``, ``"no"``, ``"f"``, ``"false"``, ``"off"`` and ``"0"``
are considered to be "false" values, and other values are considered to be invalid.
The default is for an application's snapshotting functionality to be not enabled.

.. code:: python

    import os

    os.environ["IS_SNAPSHOTTING_ENABLED"] = "y"
    application = Worlds()

    world_id = application.create_world()

    application.make_it_so(world_id, "dinosaurs")
    application.make_it_so(world_id, "trucks")
    application.make_it_so(world_id, "internet")

    application.take_snapshot(world_id)

The snapshots are stored separately from the domain events. The application
object has a ``snapshots`` attribute, which holds an event store dedicated
to storing snapshots. The snapshots can be retrieved from the snapshot store.

.. code:: python

    snapshots = application.snapshots.get(world_id, desc=True, limit=1)

    snapshots = list(snapshots)
    assert len(snapshots) == 1
    snapshot = snapshots[0]

    assert snapshot.originator_id == world_id
    assert snapshot.originator_version == 4

.. _Persistence:

Configuring persistence
=======================

The example above uses the application's default persistence infrastructure.
By default, the application object uses the library's "plain old Python objects"
:ref:`infrastructure factory <Factory>`, which provides the application
with infrastructure classes that simply keep stored events in a data structure
in memory.

To use alternative persistence infrastructure, you will need to
set the environment variable ``INFRASTRUCTURE_FACTORY`` to the
:ref:`topic <Topics>` of another infrastructure factory object
class that will construct alternative application persistence objects.
Using alternative persistence infrastructure will normally involve
setting particular environment variables that configure access to
a real database, such as a database name, a user name, and a password.

The example below shows how to configure the application to use the library's
:ref:`SQLite infrastructure <SQLite>`. In the case of the library's SQLite factory,
the environment variable ``SQLITE_DBNAME`` must be set to a file path. And if the
tables already exist, the ``CREATE_TABLE`` may be set to a "false" value (``"n"``,
``"no"``, ``"f"``, ``"false"``, ``"off"``, or ``"0"``). The function
:func:`~distutils.utils.strtobool` from the Python :mod:`distutils.utils`
module is used to interpret the value of this environment variable.

.. code:: python

    from tempfile import NamedTemporaryFile

    tmpfile = NamedTemporaryFile(suffix="_eventsourcing_test.db")
    tmpfile.name

    os.environ["INFRASTRUCTURE_FACTORY"] = "eventsourcing.sqlite:Factory"
    os.environ["SQLITE_DBNAME"] = tmpfile.name
    application = Worlds()

    world_id = application.create_world()

    application.make_it_so(world_id, "dinosaurs")
    application.make_it_so(world_id, "trucks")
    application.make_it_so(world_id, "internet")

    application.take_snapshot(world_id, version=2)


By using a file on disk, the named temporary file ``tmpfile`` above,
the state of the application will endure after the application has
been reconstructed. The database table only needs to be created once,
and so when creating an application for an already existing database
the environment variable ``CREATE_TABLE`` may be set to a "false"
value (``"n"``, ``"no"``, ``"f"``, ``"false"``, ``"off"``, ``"0"``).

.. code:: python

    os.environ["INFRASTRUCTURE_FACTORY"] = "eventsourcing.sqlite:Factory"
    application = Worlds()

    history = application.get_world_history(world_id)
    assert history[0] == "dinosaurs"
    assert history[1] == "trucks"
    assert history[2] == "internet"


Registering custom transcodings
===============================

The application's persistence mechanism serialises the domain events,
using the library's transcoder. If your aggregates' domain event objects
have objects of types that are not already supported by the transcoder,
for example custom value objects, custom :ref:`transcodings <Transcodings>`
for these objects will need to be implemented and registered with the application's
transcoder.

The application method :func:`~eventsourcing.application.Application.register_transcodings`
can be extended to register custom transcodings for custom
value objects used in your application's domain events.
The library's application base class registers transcodings
for :class:`~uuid.UUID`, :class:`~decimal.Decimal`, and
:class:`~datetime.datetime` objects.

For example, to define and register a :class:`~eventsourcing.persistence.Transcoding`
for the Python :class:`~datetime.date` class, define a class such as the
``DateAsISO`` class below, and extend the application
:func:`~eventsourcing.application.Application.register_transcodings`
method by calling the ``super()`` method with the given ``transcoder``
argument, and then the transcoder's :func:`~eventsourcing.persistence.Transcoder.register`
method once for each of your custom transcodings.

.. code:: python

    from datetime import date
    from typing import Union

    from eventsourcing.persistence import Transcoder, Transcoding


    class MyApplication(Application):
        def register_transcodings(self, transcoder: Transcoder):
            super().register_transcodings(transcoder)
            transcoder.register(DateAsISO())


    class DateAsISO(Transcoding):
        type = date
        name = "date_iso"

        def encode(self, o: date) -> str:
            return o.isoformat()

        def decode(self, d: Union[str, dict]) -> date:
            assert isinstance(d, str)
            return date.fromisoformat(d)


Encryption and compression
==========================

Application-level encryption is useful for encrypting the state
of the application "on the wire" and "at rest". Compression is
useful for reducing transport time of domain events and domain
event notifications across a network and for reducing the total
size of recorded application state.

The library's :class:`~eventsourcing.cipher.AESCipher` class can
be used to encrypt stored domain events. The Python :mod:`zlib` module
can be used to compress stored domain events. It is encapsulated
by the library's :class:`~eventsourcing.compressor.ZlibCompressor`
class.

To enable encryption and compression, set the
environment variables ``CIPHER_TOPIC`` (a :ref:`topic <Topics>`
to a cipher class), ``CIPHER_KEY`` (a valid encryption key),
and ``COMPRESSOR_TOPIC`` (:ref:`topic <Topics>` for a compressor
class).

When using the library's :class:`~eventsourcing.cipher.AESCipher` class,
you can use its static method :func:`~eventsourcing.cipher.AESCipher.create_key`
to generate a valid encryption key.


.. code:: python

    import os

    from eventsourcing.cipher import AESCipher

    # Generate a cipher key (keep this safe).
    cipher_key = AESCipher.create_key(num_bytes=32)

    # Configure cipher key.
    os.environ["CIPHER_KEY"] = cipher_key

    # Configure cipher topic.
    os.environ["CIPHER_TOPIC"] = "eventsourcing.cipher:AESCipher"

    # Configure compressor topic.
    os.environ["COMPRESSOR_TOPIC"] = "eventsourcing.compressor:ZlibCompressor"


.. _Saving multiple aggregates:

Saving multiple aggregates
==========================

In many cases, it is both possible and very useful to save more than
one aggregate in the same atomic transaction. The example below continues
the example from the discussion of :ref:`namespaced IDs <Namespaced IDs>`
in the previous section. The aggregate classes ``Page`` and ``Index`` are
defined in that section.

..
    from dataclasses import dataclass
    from uuid import uuid5, NAMESPACE_URL

    from eventsourcing.domain import Aggregate, AggregateEvent, AggregateCreated


    class Page(Aggregate):
        def __init__(self, name: str, body: str, **kwargs):
            super(Page, self).__init__(**kwargs)
            self.name = name
            self.body = body

        @classmethod
        def create(cls, name: str, body: str = ""):
            return cls._create(
                id=uuid4(),
                event_class=cls.Created,
                name=name,
                body=body
            )

        class Created(AggregateCreated):
            name: str
            body: str

        def update_name(self, name: str):
            self.trigger_event(self.NameUpdated, name=name)

        class NameUpdated(AggregateEvent):
            name: str

            def apply(self, page: "Page"):
                page.name = self.name


    class Index(Aggregate):
        def __init__(self, ref, **kwargs):
            super().__init__(**kwargs)
            self.ref = ref

        @classmethod
        def create_id(cls, name: str):
            return uuid5(NAMESPACE_URL, f"/pages/{name}")

        @classmethod
        def create(cls, page: Page):
            return cls._create(
                event_class=cls.Created,
                id=cls.create_id(page.name),
                ref=page.id
            )

        class Created(AggregateCreated):
            ref: UUID

        def update_ref(self, ref):
            self.trigger_event(self.RefUpdated, ref=ref)

        class RefUpdated(AggregateEvent):
            ref: UUID

            def apply(self, index: "Index"):
                index.ref = self.ref


We can define a simple wiki application, which creates named
pages. Pages can be retrieved by name. Names can be changed
and the pages can be retrieved by the new name.

.. code:: python

    class Wiki(Application):
        def create_page(self, name: str, body: str) -> None:
            page = Page.create(name, body)
            index = Index.create(page)
            self.save(page, index)

        def rename_page(self, name: str, new_name: str) -> None:
            page = self.get_page(name)
            page.update_name(new_name)
            index = Index.create(page)
            self.save(page, index)
            return page.body

        def get_page(self, name: str) -> Page:
            index_id = Index.create_id(name)
            index = self.repository.get(index_id)
            page_id = index.ref
            return self.repository.get(page_id)


Now let's construct the application object and create a new page (with a deliberate spelling mistake).

.. code:: python


    wiki = Wiki()

    wiki.create_page(name="Erth", body="Lorem ipsum...")


We can use the page name to retrieve the body of the page.

.. code:: python

    assert wiki.get_page(name="Erth").body == "Lorem ipsum..."


We can also update the name of the page, and then retrieve the page using the new name.

.. code:: python

    wiki.rename_page(name="Erth", new_name="Earth")

    assert wiki.get_page(name="Earth").body == "Lorem ipsum..."


The uniqueness constraint on the recording of stored domain event objects combined
with the atomicity of recording domain events means that name collisions in the
index will result in the wiki not being updated.

.. code:: python

    from eventsourcing.persistence import RecordConflictError

    # Can't create another page using an existing name.
    try:
        wiki.create_page(name="Earth", body="Neque porro quisquam...")
    except RecordConflictError:
        pass
    else:
        raise AssertionError("RecordConflictError not raised")

    assert wiki.get_page(name="Earth").body == "Lorem ipsum..."

    # Can't rename another page to an existing name.
    wiki.create_page(name="Mars", body="Neque porro quisquam...")
    try:
        wiki.rename_page(name="Mars", new_name="Earth")
    except RecordConflictError:
        pass
    else:
        raise AssertionError("RecordConflictError not raised")

    assert wiki.get_page(name="Earth").body == "Lorem ipsum..."
    assert wiki.get_page(name="Mars").body == "Neque porro quisquam..."


A more refined implementation might release old index objects
when page names are changed so that they can be reused by other
pages, or update the old index to point to the new index, so that
redirects can be implemented.


..
    #Todo: Register custom transcodings on transcoder.
    #Todo: Show how to use UUID5.
    #Todo: Show how to use ORM objects for read model.

Classes
=======

.. automodule:: eventsourcing.application
    :show-inheritance:
    :member-order: bysource
    :members:
    :special-members:
    :exclude-members: __weakref__, __dict__

.. automodule:: eventsourcing.cipher
    :show-inheritance:
    :member-order: bysource
    :members:
    :special-members:
    :exclude-members: __weakref__, __dict__

.. automodule:: eventsourcing.compressor
    :show-inheritance:
    :member-order: bysource
    :members:
    :special-members:
    :exclude-members: __weakref__, __dict__
