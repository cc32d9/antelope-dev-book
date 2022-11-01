# Data design

Whenever you build a scalable application that needs to handle lots of data with efficiency, data design becomes an important topic. As an Antelope contract developer has to face a number of limitations caused by the nature of blockchain, a proper data design becomes even more important.

A few typical questions one has to answer when designing a new smart contract:

* What is necessary to be stored in the contract data, and what can be stored outside of the blockchain.
* Which queries the contract will need to run in order to find the data elements. We will need to organize indexes for such queries.
* Which queries will the blockchain clients execute to find the data that they need. We may need to add indexes for those queries, or figure out an off-chain method of retrieving such data.
* How to organize the indexes in such a way that they actually work.
* Is the data model complete? As explained further in this chapter, modifying the data structures is an expensive process.&#x20;

## Planning the data structures

While planning the structure of your data tables, you need to take a few aspects into consideration:

* Memory and CPU are at a cost, so you only need to store the essential data that is needed for your smart contract and the application. Bulky data that is only used outside of the blockchain can be referred to by a sha256 hash or IPFS URL, and stored somewhere else.
* When reading a table row, there are always three stages:&#x20;
  * finding the table by name, [scope ](data-design.md#table-scope)and contract in nodeos state memory: logarithmic complexity
  * finding the row within the table by primary or secondary index: logarithmic complexity. But if secondary index is involved, we first retrieve the primary key from the secondary index, and then retrieve the row by primary key.
  * reading the row from nodeos state memory and deserializing it within the contract: linear complexity, proportional to the size of the entry.
* Writing a table row has approximately the same complexity as reading, but with slightly higher complexity because of secondary indexes: the row needs to be serialized, then placed in nodeos state memory with the given primary key, and then all secondary indexes need to be updated.
* Deleting a row is also not cheap, because the multi-index iterator needs to load the whole row first. Then it reads the next row in the iterator, then deletes the row and secondary indexes, and returns the iterator pointing to the next record. So, it's twice as expensive as reading a row.
* There's a RAM overhead for internal nodeos structures for each row, about 100 bytes for primary index, and about 100 bytes for secondary indexes. Also the first row in the table has to allocate more space inside nodeos for the tree structure. The best way to estimate your RAM costs is to run the contract on a testnet and measure the consumption.

Antelope allows defining the table fields as vectors. A vector is basically a linear array of data elements of the same type. It's pretty convenient for storing a series of elements that you would need within the same action execution. But there's a catch: the whole vector has to be processed every time you read or write a row of data in a table. If it's not limited in size, you may end up in a situation that 30ms time limit for transaction execution is not enough to read the whole vector. Also the multi-index implementation in CDT does not allow you to delete a row without reading it first. So, vectors need to be looked at with caution: they only make sense if the length is guaranteed to stay within reasonable limits. Otherwise you need to refactor your data structure and keep every element in a separate table row.

## Designing the indexes

Each contract table must have an `uint64_t` unique primary index.  The software wouldn't allow you insert two rows with the same primary index value. 64 bits produce 18 _billions of billions_ combinations, which is a huge number for any practical use.&#x20;

Both primary and secondary indexes are defined as functions of the table rows. In most cases those functions just return the value of one field, but they don't have to. They can calculate the index value in some more complex way, depending on the task.

A few typical ways to define the primary index in your table:

* using the account name as primary index, if your table must only have one row per account.
* &#x20;auto-incrementing values using `available_primary_key()` method in multi-index class. It returns the integer higher than the primary key of any existing row. Be aware that if the topmost row is deleted, this method will reuse its primary key.&#x20;
* auto-incrementing values using an integer value that you store in some table and increment on every new row. This will guarantee that the IDs will never be reused.
* first 8 bytes of a sha256 hash from some values in a row. This will not guarantee the uniqueness, but the chance of collision is very rare.&#x20;
* first 8 bytes of the transaction ID that inserts a row in the table. The benefit of such a method is that the client which signed the transaction already knows where the record is stored, and can query it for further processing. Keep in mind that JavaScript integer can only handle a bit more than 48 bits, so you either take 6 bytes for the index, or use BigInt. But 6 bytes is already a lot (hundreds of thousands of billions).
* the primary index can refer to a primary index in some other table. For example, table A contains long rows with bulky data which are created once and never altered, and table B contains short rows with attributes of A's rows which change frequently. This would save the CPU time of A's serialization at a cost of extra RAM.

A secondary index provides a way to find the rows by some additional criteria. In most cases it's just the value of some field in the row structure. But it can also be a more complex function of the row fields, such as a hash or a conditional.

It is important to understand that if a secondary index value refers to too many rows, it becomes unpractical or impossible to use. Let's look at an example: a table documents every student in the city, and school\_id identifies which school the student is attending:

```
  struct [[eosio::table("students")]] student_row {
    name            account;     // student's personal account
    uint32_t        school_id;   // school identifier
    auto primary_key()const { return account.value; }
    uint64_t by_school_id()const { return school_id; }
  };

  typedef eosio::multi_index<
    name("students"), student_row ,
    indexed_by<name("school"), const_mem_fun<student_row, 
               uint64_t, &student_row::by_school_id>>,
    > students;
```

If you try iterating through all students attending a given school, your request will inevitably fail: the transaction has only 30ms to finish, and HTTP API requests are limited by 10ms hard limit. So, you can only iterate through a few students, and next time you initiate the iterator, it will be pointing again to the first student in this school.

So, in this example, the secondary index needs to meet two criteria: you need a quick way to point to the first record that corresponds to a school, and you need a way to iterate through as many rows as required, in multiple steps. You need to be able to memorize the value of secondary index, so that you start walking it from the last processed position. In this example, a 128-bit index would help:

```
  struct [[eosio::table("students")]] student_row {
    name            account;     // student's personal account
    uint32_t        school_id;   // school identifier
    auto primary_key()const { return account.value; }
    uint128_t by_school_and_student()const { 
      return ((uint128_t)school_id << 64)|(uint128_t)account.value) ; }
  };

  typedef eosio::multi_index<
    name("students"), student_row ,
    indexed_by<name("school"), const_mem_fun<student_row, 
               uint128_t, &student_row::by_school_and_student>>,
    > students;
```

Here the 128-bit "school" index contains the school ID in the upper 64 bits and a copy of the primary key in the lower 64 bits. So, every row has a unique value of the secondary index, and whenever we walk through it, we can return to where we left it by using `lower_bound()` __ method in the index class.

One may notice that 18 billions of billions combinations is a bit too much for a student ID. A 32-bit integer can address 4 billion combinations, and even that would be too much for all students in any country. So, this table could be designed in a more compact way, using `uint32_t` for the primary key, and then two 32-bit integers would fit into a 64-bit secondary index. If every student has to have their own blockchain account, we could add the account field and an additional secondary index in our "students" table, as shown below.  Then the action that adds a record will need to check the uniqueness of the secondary index.

```
  struct [[eosio::table("students")]] student_row {
    uint32_t        student_id;  // unique student ID, managed by some authority
    name            account;     // student's personal account
    uint32_t        school_id;   // school identifier
    auto primary_key()const { return account.value; }
    uint64_t by_school_and_student()const { 
      return ((uint64_t)school_id << 32)|(uint64_t)student_id) ; }
    uint64_t by_account()const { return account.value ; }
  };

  typedef eosio::multi_index<
    name("students"), student_row ,
    indexed_by<name("school"), const_mem_fun<student_row, 
               uint64_t, &student_row::by_school_and_student>>,
    indexed_by<name("account"), const_mem_fun<student_row, 
               uint64_t, &student_row::account>>,
    > students;
```

## Choosing secondary indexes

As you design a new smart contract, it is tempting to add secondary indexes for every possible query. But the indexes are occupying a quite significant amount of memory, so it might turn out too expensive to maintain all the secondary indexes for millions of records in our table.

If you foresee a table with hundreds of thousands or millions of rows, you need to keep the secondary indexes down to the necessary minimum, as the RAM costs may become prohibitive. A common approach is to only define the indexes which are required by the smart contract itself. If you need the user interface to find table records quickly, there are ways to index them outside of the smart contract. For example, by using the state history plugin, one may store and update all the table records in a database which calculates all the necessary indexes automatically.&#x20;

On the contrary, a table with data which is guaranteed to occupy only a limited space may benefit from indexing the fields which are not strictly required to be indexed by the contract itself. But the user interface would be able to retrieve the records quickly without any additional infrastructure.

## Table scope

Table names and their primary indexes form a two-dimensional data space. But there is a third dimension, called _scope_. In early EOSIO design, scopes were thought to be used for parallelizing the transaction execution, but it was considered too complex later. Still, the scopes remained.&#x20;

Scope is defined by a 64-bit integer. If your contract needs only one instance of the table, you can set the scope to zero, or to the account name of the contract. Both approaches are valid, and the current trend is to use zero for the scope.&#x20;

Scopes may also be used if you need to build complex relations between tables. The table can be partitioned by scopes into a set of smaller tables, and the primary keys may overlap in them. One limitation though, is that a smart contract cannot iterate through table scopes. The scope value needs to be known at the moment of accessing the table.

The standard `eosio.token` contract uses a separate scope for each account's balance. This is a legacy design with parallel transactions in mind, and now it is a de-facto standard for a fungible token. The drawback of such a design is that every account record occupies more than 200 bytes in memory because of the overhead associated with the first row in a table (and here every scope has a single-row table).

## Querying the tables

When accessing a table from within a smart contract, it's worth remembering how the data is stored in memory. You are manipulating with multi-index objecs, similar to those in Boost C++ library, and the iterator object provides access to a deserialized table row. Whenever we need the iterator to point to a different row, it retrieves the serialized row by using low-level functions as described above, and deserializes it using the compiled-in structure definition. The multi-index object is also maintaining the internal cache, so if you re-read the same row, the retrieval and deserialization steps are skipped.

Both primary and secondary indexes provide the same set of operations for finding the row that you need. Also in both cases the index is numerically sorted from lowest to highest value. The index class provides the lookup methods as follows:

* `begin()` returns an iterator pointing to the item with the lowest key in the index. If the index is empty, the returned value is equal to `end()`.
* `end()` returns an iterator pointing the the item _next after the highest one_. So it's an invalid item, and trying to access the data at this iterator will result in an exception error.
* `find(key)` and `require_find(key, errmsg)`look up for an element with exact match on the key value. The second form is simply a combination of `find()` and `check()` calls. In case of a secondary non-unique index, there's no guarantee about which of the matching elements is returned, so it's not suitable for traversing several values.
* `get(key, errmsg)` does not return an iterator, but a reference to the row structure. One may find it more convenient than `find()`, although there's not much difference.
* `lower_bound(key)` is used for accessing the indexes where you don't know the exact key value. It returns the first (lowest) element that is equal or bigger than the specified key value. Whenever you need to iterate through a range of table rows, you typically start the loop from the lower bound, and on every incrementing of the iterator you check if the returned row is still within the desired range. An example will follow.
* `upper_bound(key)` has caused a lot of confusion because of an error in EOSIO documentation. The best reference is the definition from the Boost library which implements this call in nodeos: [_returns an iterator pointing to the first element with key greater than `x`, or `end()` if such an element does not exist_](https://www.boost.org/doc/libs/1\_36\_0/libs/multi\_index/doc/reference/ord\_indices.html). Most of the time you would only use the `lower_bound` call, and pretty rarely you would need the `upper_bound`.



## Altering the table structure

As explained above, a table row is a vector of bytes, serialized by the contract. Also, the serialization format is not flexible enough to allow modifying the structure and be able to read the old rows. Also the secondary indexes are only updated on inserting or modifying the rows.

So, if you really need to modify the fields in a table, or modify a secondary index definition, it can only be done on an empty table if you're using the standard CDT classes. It's still possible to implement a low-level iterator which would scan the table and apply changes in each row, but this would render the table unusable for the time of migration.&#x20;

If a migration contract rewrites a table (for example, by copying each row into a new table and erasing the old row), the information about original RAM payers is lost, so the migrated data rows would have the admin account or the contract itself as a RAM payer.&#x20;

Sometimes it is sufficient to just create an additional table which would supply additional data for the same primary index values as in the original table. The drawback is that an additional row lookup will require more CPU time.

## Storing binary data as binary

Quite often a contract needs to store data that is originally in binary form, but there is a common user-friendly representation (such as IPFS links, Ethereum or Bitcoin addresses, etc.). If a contract stores this data in this user-friendly text representation, the amount of used RAM is significantly higher than necessary (for example, a hexadecimal string of an Ethereum address, with `0x` prefix, is 42 bytes long. But it represents 20 bytes of the actual address). In addition, if a contract processes this data somehow, it will have to spend the CPU time on string conversion.

It always makes sense to store the binary data in its original binary form in a vector of bytes:   `std::vector<uint8_t>` , which is reflected by `bytes` type in the ABI. The HTTP API in nodeos will convert such data to and from hexadecimal string format automatically (although it won't contain the `0x` prefix).

It is worth reminding that SHA-256 is not the only possible way to address IPFS files, and the [format is well described](https://richardschneider.github.io/net-ipfs-core/articles/multihash.html).
