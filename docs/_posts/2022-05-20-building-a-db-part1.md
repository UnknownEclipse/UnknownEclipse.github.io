---
layout: post
title: "Building a Database, Part 1"
date: 2022-05-20 10:21:48 -0700
categories: jekyll update
---

Why? Good question!

...

Anyway, let's get started! I'll be using Rust on a MacBook, but I'll make sure to point
out any platform specific quirks to handle as necessary.

Now, to figure out how databases actually work. The general principle behind all database
engines is that they are designed to store and organize data extremely reliably, and
perform queries that operate on that data. Reliability is extremely important, so let's
start with that.

Database writes must be atomic, meaning that even if an error occurs while we are making
a change, none of the changes are visible and the database can be easily restored. This
is quite a challenge, [especially when filesystems are rude enough to have bugs](http://danluu.com/file-consistency/).

The reliability aspect is handled by the storage layer, which turns abstract requests
(write this data here, read this data there) into efficient and reliable filesystem
operations. Let's start by defining our storage layer's interface, then I'll go
more into depth on the choices made.

```rust
pub struct Store {
    // ...
}

pub struct Reader<'store> {
    // ...
}


pub struct Writer<'store> {
    // ...
}

pub struct PageId(u32);

pub struct PageRef<'a>(/* ... */);

impl Store {
    pub fn reader(&self) -> Reader<'_> {
        todo!()
    }

    pub fn writer(&self) -> Writer<'_> {
        todo!()
    }
}

impl<'store> Reader<'store> {
    pub fn get(&self, page: PageId) -> Result<Option<PageRef<'_>>> {
        todo!()
    }
}

impl<'store> Writer<'store> {
    pub fn get(&self, page: PageId) -> Result<Option<PageRef<'_>>> {
        todo!()
    }

    pub fn write(&self, start: usize, data: &[u8]) -> Result<()> {
        todo!()
    }

    pub fn commit(self) -> Result<()> {
        todo!()
    }

    pub fn rollback(self) -> Result<()> {
        todo!()
    }
}
```

The core of our storage layer is the `Store` struct. On its own, it can't do much.
We don't allow reads or writes to occur directly on the `Store`. Instead, we have
methods that return `Reader`s and `Writer`s which allow those operations. Both of these
methods work on _snapshots_ of the database, which is important. Imagine if, at the same
time as you tried to read some data, that same data was being rearranged by a writer.
Chaos ensues! (This is also what Rust's borrow checker stops).

`Reader`s only have one method: `get()`, which returns a page with the given id if it
exists. Pretty simple. `Writer`s on the other hand, have a whole bunch of other methods.
The most basic here is `write()`, which writes some data into a page at the requested
offset. The next two are `commit()` and `rollback()`. These two are related. The idea
is that, once the database is done with a write operation, those writes can be atomically
'committed', or made visible to readers and essentially become part of the database.
Until that data is committed, readers will continue to view the database before any
of those writes occurred. A rollback does the opposite, essentially saying, "hey, yeah
I know I did all that writing, but what if you like... forgot." The writes made during
that write transaction will be forgotten.
