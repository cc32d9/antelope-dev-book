# Smart contract memory

## Antelope contract data store

We are used to talking about smart contract _tables_, although in fact they are binary trees: the lookup keys are stored in an ordered set, and whenever we need to find an element we perform a binary search.

It also means we can't efficiently count the keys within a range: in order to count them, we need to visit each key.

In order to better understand the data model, it's useful to look inside `multi_index.hpp` which is part of Antelope CDT, and can be found inside `/usr/opt/eosio.cdt/1.8.1/include/eosiolib/contracts/eosio/` if you installed the CDT from a package (the version may vary). In general, it is a good practice to look inside the header files that come with CDT if you need to understand how things work.

The most essential part of smart contract data interaction is defined in a place with a funny name: `namespace internal_use_do_not_use`. Inside it, we see a number of _intrinsics_, or functions exported by `nodeos` into the WASM virtual machine. Unfortunately the header does not explain those lower-level functions, so you need to look at their counterparts in `libraries/chain/include/eosio/chain/webassembly/interface.hpp` in `nodeos` sources. What these functions define is in fact the low-level access to contract data which is managed by `nodeos`.&#x20;

The function that creates a row in a contract table is a good illustration of what's going on:

```
 int32_t db_store_i64(uint64_t scope, uint64_t table, 
                      uint64_t payer, uint64_t id, 
                      legacy_span<const char> buffer);
```

As you can see, the value of a row is an abstract vector of bytes (so, the node does not know anything about the structure stored in it), and the row is uniquely found by a combination of code, scope, table name, and the primary key value (`db_store_i64` does not have the code (the contract account), because only the contract executing an action is allowed to update its tables. But you can see the whole quadruple in `db_find_i64` and `db_lowerbound_i64`. The functions return a 32-bit iterator which allows retrieving the content of the row (see `db_get_i64`).

Further on in `interface.hpp` you can find the functions for secondary index access (such as `db_idx64_store`. You may notice two important things:

* Secondary indexes are updated separately from table rows. The C++ multi-index class  in CDT is hiding this fact from the developer by providing convenient wrappers which update the secondary indexes whenever a row is updated.
* If you look at the list of arguments, each function that manages a secondary index can only refer to one index of each type. But we know that we can create  multiple indexes of the same type. The trick is that the `multi_index` class uses the upper 4 bits in table name for creating multiple indexes of the same type for a given table. That's why the table name can never consist of 13 symbols, and 12 symbols is the maximum.

Further down in `multi_index.hpp`  we can see the  `multi_index` class definition which wraps those low-level calls into a convenient programming interface.

You may notice a few interesting facts by studying this header file:

1. A table row is always retrieved from the store and deserialized, whenever you access it via any of the class methods or the iterator.
2. But not always, because there's cache: the multi-index object memorizes the retrieved row and returns it on the next read. This speeds up the operation if you access the same row several times. But if two indexes of multi-index are referring to the same table, modifying or deleting a row may result in unwanted effects. The **rule of thumb** is to avoid creating multiple multi-index objects for the same table within the same action execution.
3. Whenever you erase a row, the next row  in the iterator is retrieved from the contract store. So, erasing a row is not a cheap operation: even if you delete one row in an action, at least two rows have been retrieved when the action is executed.

Also, food for thought for a curious reader:

* The data in a contract table doesn't _have to_ be serialized in the standard way. The node doesn't really care what inside that vector of bytes. But the HTTP API and history indexes do care, of course. So, the developer may want to choose a different way of storing data.
* Partial indexes are theoretically possible, but they need a different class, as the standard multi-index creates secondary index entries automatically for every row. For example, your table contains a boolean flag, and you are only interested in accessing the rows where this flag is true. In theory, you can save on RAM significantly if the secondary index is only covering the rows where this flag is set. The only issue is that there's no battle-tested library which would implement it.

## RAM payers and quotas

Every low-level operation which requires a new memory allocation is taking a `payer` parameter. The node keeps track of ownership of every table row, and deducts the alocation size from the payer's quota.&#x20;

`Nodeos` allows only the contract itself or an accout which authorized the transaction to be the RAM payer.

Each account has a quota of RAM that it's allowed to spend on data structures. Also privileged accouts (such as the system contract `eosio`) have unlimited RAM quotas.

It is up to the system contract to define how the quotas are assigned to accounts. In most blockchains RAM needs to be purchased on the RAM market.
