# lazers-traits - laze RS interface

## General philosophy

This library models the general interactions with CouchDB-like storages,
be it a CouchDB server itself or a local K/V store with a CouchDB interface.

### Techniques

* All operations return Results. An Error value describes a _failed
  interaction_, not a negative query result (such as a database missing).

* Responses with multiple semantic meanings are mapped to enums.
* The library uses decorations of Result types and these enums for easier
access.

## Dependencies

We use `serde`s definitions for serialisation/deserialisation.

serde provides many features we want, including the ability to read
documents
in a typesafe manner.

```rust
extern crate serde;
```

We use the error chain macro to provide the ability to wrap external errors
in an easy fashion.

```rust
#[macro_use]
extern crate error_chain;
```

To express asynchronicity, we use futures-rs.

```rust
extern crate futures;
```

## Exports

The library exports two modules.

[`result`](/lazers-traits/src/result) defines our own `Result` type. See the module page for details.

[`prelude`](/lazers-traits/src/prelude) exports all definitions needed for day-to-day work, this allows
users
to simply `use lazers_traits::prelude::*` instead of loading a huge block of
codes imports themselves.

[`decorations`](/lazers-traits/src/decorations) collects all convenience decorations of the library on, for
example, `Result` types.

```rust
pub mod result;
pub mod prelude;
pub mod decorations;
```

## Use of externals

We use a custom error created using error_chain!.

```rust
use result::Error;
```

Futures, we use mainly through the BoxFuture interface. This carries with it the information that we expect all futures to be `Send`.

```rust
use futures::BoxFuture;
```

We don't implement our own `Deserialize` and `Serialize` traits, but instead
use the ones from serde.

```rust
use serde::de::Deserialize;
use serde::ser::Serialize;
```

We have to provide custom `Debug` implementations, so we import the trait.

```rust
use std::fmt::Debug;
```

## Definitions

### DatabaseName

The DatabaseName is anything we can use to name a database. Currently, this
type is just an alias for String.

```rust
pub type DatabaseName = String;
```

### Document

CouchDB is all about handling documents, which means we have to find a
definition for what constitutes a document. In our case, we decide that
anything that can be serialised and deserialised by serde is a document.

Also, we provide a blanket implementation that ensures that every type that
is Deserialize and Serialize.

The Document trait is a marker trait and holds no methods.

Documents, as a design choice, don't hold information about the database
they were loaded from.

Finally, all Documents must be `Send`, as the represent plain data.

```rust
pub trait Document: Deserialize + Serialize + Send {}

impl<D: Deserialize + Serialize + Send + ?Sized> Document for D {}
```

### Key

Keys are the main method of addressing Documents in CouchDB. As keys can
take
many forms and are regularly used to encode data, we only express the bare
minimum as a trait.

Keys also encode the revision of the current document. The revision is
optional, but must be given for documents already in the database.

Along with the trait definition, we ship the most basic implementation of it
for users to use, a simple struct with a `String` key and an optional `rev`
`String`.

```rust
pub trait Key: Eq + Clone + Debug + Send {
    fn id(&self) -> &str;
    fn rev(&self) -> Option<&str>;
    fn from_id_and_rev(id: String, rev: Option<String>) -> Self;
}

#[derive(Debug,Clone,PartialEq,Eq)]
pub struct SimpleKey {
    pub id: String,
    pub rev: Option<String>,
}

impl Key for SimpleKey {
    fn id(&self) -> &str {
        &self.id
    }

    fn rev(&self) -> Option<&str> {
        match self.rev {
            Some(ref string) => Some(string),
            None => None,
        }
    }

    fn from_id_and_rev(id: String, rev: Option<String>) -> Self {
        SimpleKey { id: id, rev: rev }
    }
}

impl From<String> for SimpleKey {
    fn from(string: String) -> SimpleKey {
        SimpleKey {
            id: string,
            rev: None,
        }
    }
}
```

### The Client Trait

The client trait is the entry point to all global storage level operations
of
CouchDB. Mostly, this is querying for named databases.

Other operations are currently not supported.

All operations return a result.

```rust
pub trait Client: Default {
    type Database: Database;

    fn find_database(&self, name: DatabaseName) -> BoxFuture<DatabaseState<Self::Database, <<Self as Client>::Database as Database>::Creator>, Error>;
    fn id(&self) -> String;
}
```

### The DatabaseState Enum

Querying for a database by name returns an enum describing two possible
options:

1. The database exists. A handle to the database can be retrieved from the
`Existing` variant.

2. The database is absent. In this case, the `Absent` variant holds the
handle
to a `DatabaseCreator`. The Creator can then be used to create the database.

For simple querying, `existing` and `absent` methods are implemented.

```rust
pub enum DatabaseState<D: Database, C: DatabaseCreator> {
    Existing(D),
    Absent(C),
}

impl<D: Database, C: DatabaseCreator> DatabaseState<D, C> {
    pub fn absent(&self) -> bool {
        match self {
            &DatabaseState::Absent(_) => true,
            _ => false,
        }
    }

    pub fn existing(&self) -> bool {
        !self.absent()
    }
}
```

## The DatabaseCreator

A DatabaseCreator trait describes the creation of a database of a _known_
name.

It does not provide a way to create a database by passing a name, as it is
intended for use with the DatabaseState enum only. Implementors should pass
the
name of the database to be created to the underlying structure.

```rust
pub trait DatabaseCreator
    where Self: Sized + Send
{
    type D: Database;

    fn create(self) -> BoxFuture<Self::D, Error>;
}
```

### The `Database` trait

The `Database` trait describes one `database` in CouchDB lingo. A database
is a
seperate key-value bucket, holding documents and design documents.

### Lifecycle

A struct implementing the `Database` trait also allows destroying the
database,
which also deletes all documents along with it.

Destroying the database is a consuming operation, returning a
`DatabaseCreator`
on success, to allow creating it again if wanted.

### DatabaseInfo

DatabaseInfo provides access to several pieces of data a `CouchDB`-like database _must_ implement. Several clients might give richer information
for which they should build additional interfaces.

```rust
pub struct DatabaseInfo {
    instance_start_time: String,
    update_seq: UpdateSeq,
}

impl DatabaseInfo {
    pub fn new(instance_start_time: String, update_seq: UpdateSeq) -> DatabaseInfo {
        DatabaseInfo {
            instance_start_time: instance_start_time,
            update_seq: update_seq,
        }
    }

    pub fn instance_start_time(&self) -> &str {
        self.instance_start_time.as_ref()
    }

    pub fn update_seq(&self) -> &UpdateSeq {
        &self.update_seq
    }
}
```

### UpdateSeq

The CouchDB update sequence is [either a number, a string or an array of a number and a string](https://github.com/pouchdb/pouchdb/issues/3220). This complexity should be hidden from users, but put into the respective clients.

UpdateSeq is used in the replication protocol and checks if two databases have reached the same state.

```rust
#[derive(Eq, PartialEq)]
pub enum UpdateSeq {
    Numeric(u64),
    String(String),
    Pair(u64, String)
}
```

### Database access

The methods for database access are all generic over the key and the
document
type(s) retrieved. Serialisation and Deserialisation failures are expressed
as
Errors.

* `info`: retrieve general info about the database.

* `doc`: returns a handle on a database entry, described in "The
  `DatabaseEntry` enum"

* `insert`: directly inserts a document without previously retrieving
  information about it. Occuring conflicts are errors.

* `delete`: directly deletes a document without previously retrieving
  information about it. Occuring conflicts or missing necessary revision
  information results in an error.

```rust
pub trait Database
    where Self: Sized + Send
{
    type Creator: DatabaseCreator<D = Self>;
    //type DBInfo: DatabaseInfo;

    //fn info(self) -> BoxFuture<Self::DBInfo, Error>;
    fn destroy(self) -> BoxFuture<Self::Creator, Error>;
    fn info(&self) -> BoxFuture<DatabaseInfo, Error>;
    fn doc<K: Key + 'static, D: Document + 'static>(&self, key: K) -> BoxFuture<DatabaseEntry<K, D, Self>, Error>;
    fn insert<K: Key + 'static, D: Document + 'static>(&self, key: K, doc: D) -> BoxFuture<(K, D), Error>;
    fn delete<K: Key + 'static>(&self, key: K) -> BoxFuture<(), Error>;
}
```

### The `DatabaseEntry` enum

The `DatabaseEntry` enum describes the three possible states of an entry,
queried by key, in a CouchDB database:

* `Present`: There is a document for this key
* `Absent`: There is no document for this key
* `Conflicted` : There are conflicts for this key

As this information makes no sense without knowing the database the key
belongs
to, all variants of `DatabaseEntry` hold a reference to the `Database`
handle
they result from.

For all three variants, convenience constructors are provided.

An entry is considered "existing" if there's either a document for this
key, or
a conflicts. An appropriate query method is provided.

```rust
#[derive(Debug)]
pub enum DatabaseEntry<K: Key, D: Document, DB: Database> {
    Present { key: K, doc: D, database: DB },
    Absent { key: K, database: DB },
    Conflicted {
        key: K,
        documents: Vec<D>,
        database: DB,
    },
}

impl<K: Key, D: Document, DB: Database> DatabaseEntry<K, D, DB> {
    pub fn present(key: K, doc: D, database: DB) -> DatabaseEntry<K, D, DB> {
        DatabaseEntry::Present {
            key: key,
            doc: doc,
            database: database,
        }
    }

    pub fn absent(key: K, database: DB) -> DatabaseEntry<K, D, DB> {
        DatabaseEntry::Absent {
            key: key,
            database: database,
        }
    }

    pub fn exists(&self) -> bool {
        match self {
            &DatabaseEntry::Present { .. } |
            &DatabaseEntry::Conflicted { .. } => true,
            _ => false,
        }
    }
}
```

### Decorations

Standard operations over the described types are implemented as decorations
and
can be found in the `decorations` module.
