# RasgueaDB

RasgueaDB is an indexing and query layer for [LMDB](https://github.com/hoytech/lmdbxx) and [flatbuffers](https://google.github.io/flatbuffers/). It uses templates to generate a C++ header file according to your schema, which is specified as YAML. This compiled header file provides a framework for inserting, updating, and deleting records. Additionally, it supports indexing the records, and will maintain the indices whenever you perform modifications.

Because records are stored as flatbuffers inside LMDB, accessing fields results in [std::string_view](https://github.com/hoytech/lmdbxx#string_view)s that point into the LMDB memory map. Fields that you don't access are never even parsed or loaded, and the ones you do are provided as zero-copy references. Even for relatively small values that don't benefit from the zero-copy itself, an often overlooked benefit is that there are no memory allocations required to read a record.

Querying and iteration is simple but flexible. When looping over records you provide lambda callbacks. These callbacks, the wrapper code, and all database access reside in the same compilation unit, which allows the compiler to optimise it extensively.


## Synopsis

Here is a simple example schema:

    db: example

    tables:
      User:
        tableId: 1

        fields:
          - name: userName
            type: string
            unique: true
          - name: passwordHash
            type: ubytes
          - name: created
            index: true
            ## default type is uint64

* The `db` field is a name for your database. It will be used for namespacing in the generated files and the database table names.
* Each table should have a numeric `tableId`. This is used to track updates in the OpLog.
* For now there are 3 types supported:
  * `uint64`: Unsigned 64-bit integer. This is the default when no type is specified.
  * `string`: Arbitrary byte-string.
  * `ubytes`: Identical to `string`, except for how it is displayed when debugging.

After compiling the schema, here is an example program that shows how to use the generated header:

    #include <iostream>
    #include "build/example.h"

    int main() {
        example::environment env;

        env.open("/path/to/db/");

        // Populate a record

        {
            auto txn = env.txn_rw();
            env.insert_User(txn, "john", "\x01\x02\x03", 1000);
            txn.commit();
        }

        // Query a record via the userName index

        {
            auto txn = env.txn_ro();
            auto view = env.lookup_User__userName(txn, "john");
            if (!view) return -1; // not found
            std::cout << view->userName() << std::endl;
        }

        // Loop over all records

        {
            auto txn = env.txn_ro();
            env.foreach_User(txn, [&](auto &view){
                std::cout << view.userName() << std::endl;
                return true; // keep looping
            });
        }

        return 0;
    }

## Insert

Insert a new record:

    env.insert_User(txn, "john", "\x01\x02\x03", 1000);

This must be done inside a read-write transaction (this is also the case for updates and deletions).

After the `txn` transaction handle, the following arguments correspond to the fields defined for this table in your schema.

This call can throw an exception if a unique constraint is violated (see below).

## Primary keys

Every record has a primary key of type `uint64`. By default these are autogenerated when a record is inserted, starting with id `1`. The primary key of `0` is invalid.

Instead of using the auto-generated ids, you can specify a field to use as the primary key instead. This field must be of type `uint64`:

      Address:
        tableId: 2

        primaryKey: userId

        fields:
          - name: userId
          - name: street
            type: string
          - name: city
            type: string

Choosing an alternative primary key is useful for "may-have-a" relationships between tables, and to ensure that these sub-tables have the same ordering as their primary tables.

By referencing another table's primary keys, you can implement foreign-key relationships, although currently this is not enforced.


## Lookup by primary key

Here is how to lookup the record with a primary key of `2`:

    auto view = env.lookup_User(txn, 2);

`view` is of type `std::optional<View_User>`. If the record was not found, then this will be `std::nullopt`. Otherwise, it is a "pointer" to a `View_User` object which provides accessors for each field, and the primary key for the record:

    if (view) {
        std::string_view userName = view->userName();
        uint64_t primaryKey = view->primaryKey;
    } else {
        // not found
    }

## Update

Update a record (C++20 syntax):

    env.update_User(txn, view, { .userName = "new username", });

When you update, all the indices will be updated as well.

If you use the C++20 way with [designated initialisers](https://en.cppreference.com/w/cpp/language/aggregate_initialization) then the fields must be in the same order as they are defined in the schema otherwise you will get a compile-time error. If you don't want to use designated initialisers, you can do it like this:

    example::environment::Updates_User upd;
    upd.username = "new username";
    env.update_User(txn, view, upd);

As with insert, an update can fail with an exception if it causes a unique constraint to be violated.


## Delete

To delete a record, you need its primary key, which you can retrieve from a view:

    env.delete_User(txn, view->primaryKeyId);


## Unique

A `unique` modifier can be specified on an index:

        fields:
          - name: userName
            type: string
            index:
              unique: true

When you insert or update a record it will check that no other records have the same value for this index. If there are, then exception will be thrown.

By default duplicates *are* allowed, and in the index these duplicates are sorted by primary key.


## Lookup by index

If a field is indexed, then you can lookup a record like this:

    auto view = env.lookup_User__userName(txn, "alice");

Note: The table name is separated from the index name by two underscores.

If there are multiple records with the same value, then it will return the first one it finds.

## Iteration over table

To iterate over a whole table:

    env.foreach_User(txn, [&](auto &view){
        // ...
        return true;
    });

Your callback must return a `bool` which indicates whether the looping should continue. By returning false early you can implement `LIMIT`-like functionality. Note: If you forget the `return true` you'll get a inscrutable compile-time error.

The records will be iterated over in order of their primary keys.

## Iteration over index

This iterates over all the entries in an index. In this case, it will be the same as the previous example, except the records will be sorted by `userName`:

    env.foreach_User__userName(txn, [&](auto &view){
        // ...
        return true;
    });

There is an optional boolean parameter `reverse`, which signifies the index should be iterated over in reverse order:

    env.foreach_User__userName(txn, [&](auto &view){
        // ...
        return true;
    }, true);

Instead of starting at the beginning (or end, if reversing), to start at a particular record you can pass this in as the next optional parameter, `start`:

    env.foreach_User__userName(txn, [&](auto &view){
        // ...
        return true;
    }, false, "bob");

This will start the iteration at `bob` (skipping over all records with `userName` lexically preceeding `bob`). If a record with `bob` is not present, it will start the iteration at the first existing record lexically following `bob`. In the reversing case, if a starting record does not exist, the iteration will begin with the first existing record lexically preceeding your specified start. 

The combination of starting records and the "continue looping" boolean returned from your callback allows you to perform perform range-queries and pagination.


## Iteration over dup records

Unless an index is marked `unique`, there can be multiple records with the same indexed value. Here is how to loop over all records that have a particular indexed value (`1001` here):

    env.foreachDup_User__created(txn, 1001, [&](auto &view){
        // ...
        return true;
    });

Similar to iteration over and index, `reverse` and `start` parameters are accepted:

    env.foreachDup_User__created(txn, 1001, [&](auto &view){
        // ...
        return true;
    }, reverse, start);

`start` corresponds to the primary key of the record you would like to start with (which is always of type `uint64`).

There is also a `count` optional parameter. If passed, it should be a pointer to a `uint64_t`. After the iteration is done, the total number of dups will be stored in this variable (even if you stop looping prematurely).

    uint64_t total;

    env.foreachDup_User__created(txn, 1001, [&](auto &view){
        // ...
        return true;
    }, reverse, start, &total);

The `count` parameter is useful for implementing pagination. When retrieving the first page of results, you can also get the total number of results. This is slightly more efficient than computing it separately because in order to iterate you've already positioned the cursor to these dups.


## Derived indices

Although you can make a particular field indexed by adding an `index` key to the field, you can also have indices that don't directly correspond to fields. The indexed values can be created with arbitrary C++ code by including an `indexPrelude` and then a set of `indices` to add. The `accessor`s in the indices should correspond to local variables you defined in the `indexPrelude`.

As an example, here is definition of a table that maintains indices of its two fields in lower-case:

      Person:
        tableId: 2

        fields:
          - name: fullName
            type: string
          - name: email
            type: string

        indexPrelude: |
          std::string fullNameLC = std::string(v.fullName());
          std::string emailLC = std::string(v.email());
          std::transform(fullNameLC.begin(), fullNameLC.end(), fullNameLC.begin(), ::tolower);
          std::transform(emailLC.begin(), emailLC.end(), emailLC.begin(), ::tolower);

        indices:
          fullNameLC:
            accessor: 'fullNameLC'
          emailLC:
            accessor: 'emailLC'
            unique: true

Now records can be queried using the index without worrying about casing, but the original user's chosen casing is preserved.

Note that the `emailLC` index is also `unique`. This will prevent duplicated records from being inserted, even if the casing is different.

IMPORTANT: When deriving index values, given the same view (accessible as `v`) you must always compute the exact same index values, otherwise your database will become corrupted. It is recommended to only use data from `v` to compute the values (and nothing else).


## Multi-indices

A `multi` modifier can also be added to an index definition. This is for when a record should get multiple entries in the same index.

The test-suite has an example where a `words` field is split up into individual words and each is added to a multi-index. This way you can query for all records that contain a particular word somewhere in their `words` field.

For multi-indices, the variable referred to by `accessor` should be of type `std::vector<>`.



## OpLog

A special table `OpLog` is automatically added to your schema. This is a record of modifications to the DB, and is useful for streaming updates to users, and/or for replication.




## Author and Copyright

RasgueaDB © 2021 Doug Hoyte.

2-clause BSD license. See the LICENSE file.

Does this stuff interest you? Subscribe for news on my upcoming book: [Zero Copy](https://leanpub.com/zerocopy)!
