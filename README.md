# RasgueaDB

Indexing and querying layer for [LMDB](https://github.com/hoytech/lmdbxx) and [flatbuffers](https://google.github.io/flatbuffers/). Compiles a YAML schema to a C++ header file.


## Synopsis

Here is an example schema:

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

        // Query record

        {
            auto txn = env.txn_ro();
            auto view = env.lookup_User__userName(txn, "john");
            if (!view) return -1; // not found
            std::cout << view->userName() << std::endl;
        }

        // Loop over all records

        {
            auto txn = env.txn_ro();
            env.foreach_User__userName(txn, [&](example::environment::View_User &view){
                std::cout << view.userName() << std::endl;
                return true; // keep looping
            });
        }

        return 0;
    }
