# Under the hood of a smart contract

In order to be able to write efficient smart contracts, one needs to understand what is happening behind the scenes. An experienced developer will find many similarities to micro-controller or embedded device programming: there's a certain set of constraints, and your application design needs to fit into them. Your program needs to utilize the libraries and resources efficiently. It also needs to be scalable so that you're not stuck when there's 10 thousand data entries instead of a dozen that you tested before the deployment.

There are two important aspects in the smart contract operation: data handling and the execution environment.&#x20;

## Data serialization

In general, the term _serialization_ is used when a data structure is converted into a byte sequence for further _deserialization_. JSON and Google Protocol Buffers would be typical examples of serialization.

Both JSON and Protocol Buffers are very flexible and are used in many applications, but they require some complex parsing and state tracking (like curly braces and syntax elements in JSON) and error handling. Antelope is designed for fast transaction execution, so its serialization standard is limited in flexibility, but it allows a much faster processing within the smart contract.

The serialization format does not have any field delimiters, and primitive types don't contain the length field. So, both writer and reader need to have a common understanding about the data structure before starting the serialization or deserialization.&#x20;

If a C++ smart contract needs to read or write a serialized data structure, it has to have a `struct` type definition for it. The Antelope Contract Development Toolkit (CDT) provides a number of ways to  automate this process and hide it from the developer. It sometimes leads to a confusion because the developer does not understand the whole process.

By using a `[[eosio::table]]` attribute specifier in your source code, you tell the CDT to prepare the C++ methods for serializing and deserializing the structure. For example,

```
struct [[eosio::table("stock")]] stockrow {
  uint64_t       skuid;
  uint64_t       items_onsale = 0; // number of items available on sale
  time_point     last_sale;
  auto primary_key()const { return skuid; }
};

typedef eosio::multi_index<name("stock"), stockrow> stockrows;
```

This structure definition creates reading and writing methods which will convert between three serialized 64-bit integers and a structure in memory. If this byte array was written for a different structure definition, the reader method will either read garbage or fail because the number of bytes in the array is not what it expected. So, if the reader has a structure definition with a different order of fields, different types (for example, a 32-bit integer instead of 64-bit one), or different number of fields, it will not be able to read what the writer stored in that byte array.

Action definitions, although looking completely different, perform a similar job:

```
 [[eosio::action]] challenge(name challenger, uint64_t id, 
                             name account, checksum256 hash, 
                             uint32_t expire_seconds)
 {
```

Action arguments are also serialized and deserialized using the same serialization protocol, so the C++ compiler needs to know how to pack or unpack the arguments from a byte array. The `[[eosio::action]]` attribute (there is also an `ACTION` macro for simplicity) tells the compiler that the function arguments will be serialized or deserialized. In this example above, the action arguments will consist of a byte array where three 64-bit integers are followed by a 32-byte array followed by a 32-bit integer. All these values are fixed-width, so there will be no delimiter between them and no information about their length. In essence, there is no difference in handling a structure or function arguments.&#x20;

If a structure has variable-length fields, such as `std::string` or `std::vector`, the serialization protocol adds the length in the beginning of data. The length is stored as _varuint_, which is a special type of variable length, so shorter strings will have a shorter length field.

On some occasions, the CDT is unable to construct the methods for serializing complex data structures, such as those containing `eosio::public_key` type of field. In such cases, the contract code needs an explicit definition of serialization methods, using `EOSLIB_SERIALIZE` macro:

```
struct [[eosio::table("messages")]] message {
  uint64_t         id;             /* autoincrement */
  name             sender;
  vector<char>     iv;
  public_key       ephem_key;
  vector<char>     ciphertext;
  checksum256      mac;
  auto primary_key()const { return id; }
};
EOSLIB_SERIALIZE(message, (id)(sender)(iv)(ephem_key)(ciphertext)(mac));
typedef eosio::multi_index<name("messages"), message> messages;  
```

If a program outside of a smart contract needs to read or write the serialized data, it is normally using the ABI (Application Binary Interface), which is usually published alongside the smart contract. The Antelope ABI can be represented in two ways: a compact binary form, suitable for storing on the blockchain; and JSON, suitable for human and machine readers. The ABI reflects all the structures (and also action arguments as structures) that are defined in the smart contract. External programs, such as wallets or other blockchain clients, usually read the ABI first, and it allows them to read the smart contract tables content or encode the action arguments before sending a transaction.

## Action dispatcher

If you write a simple smart contract using the C++ CDT, you typically define a class and mark certain methods as actions.

However, `nodeos`, which is executing your contract, does not know anything about C++ classes and methods. All it does is call the `apply()` function (which is a C function exported by your WASM binary) within a freshly initialized WASM virtual machine and passes several arguments to it:

`void apply( uint64_t receiver, uint64_t code, uint64_t action )`

This is a C function, and the arguments are integer values, although we are used to working with `eosio::name` objects. The CDT header files convert the names to integers for us.  `receiver` is the account name that received the initial action, and `code` is the account that is executing the action at this particular moment.

If it's an action that was a transaction input, or a result of  `eosio::action::send()` execution, then `code` is equal to `receiver`. Otherwise, this call is a result of a `require_recipient()` call, and `receiver` holds the account name of the contract where `require_recipient()`  was called, so it's different from `code`.&#x20;

Basically, by comparing `code` and `receiver` you can determine if you are processing the original action call or a notification.

The `action` argument corresponds to the action name: if we are processing the result of `require_recipient()` , `action` is the name of the original action from where `require_recipient()` was called. Otherwise, it's the name of the action that was called in your contract.

So, the dispatcher within the `apply()` function needs to perform several tasks:

1. Find out if it's an action call or a notification (by comparing  `code` and `receiver`).
2. Find the handler for this action or notification.
3. Decode the action arguments and pass them to the action handler.

As you can see, the action arguments are not passed to `apply()` function directly. The execution environment provides two low-level C calls for that: `action_data_size()` tells the length of serialized arguments, and `read_action_data()` copies the serialized arguments into a provided buffer.

The serialized action arguments do not contain any information about their structure. Whatever the client has packed, is coming as a vector of bytes. Now the dispatcher needs to extract individual fields from this data and pass them to the action handler, and the only information it has is the class method definition in your contract sources.&#x20;

C++ comes in handy here with the use of _templates_. The dispatcher code is a generalized template taking the class method and using its list of arguments as a list of types that it passes to the deserializer.

Thus, if the caller has serialized the arguments in the wrong order, or used incorrect data types (for example, packing an uint64 while the dispatcher expected uint32), the action may fail before it's even passed to the handler because the dispatcher could not deserialize the arguments.&#x20;

Using the modern CDT kit, you do not need to care about the `apply()` function as the CDT is generating it from `[[eosio::action]]` and `[[eosio::on_notify]]` labels in your C++ sources.

Let's have a closer look at the `challenge` action example above. The `[[eosio::action]]` doesn't have any name specifier, so it's equal to `[[eosio::action("challenge")]]`. The name in the parens defines the action name that will be used by the dispatcher and will be published in the ABI. Then comes the class method name, which could -- in theory -- be different from the action name, but practically it's more convenient when the methods have the same names as actions.&#x20;

A notification handler may be specific for a particular contract name or a generic one using a wildcard:

```
/* specific handler reactinhg on transfers in system token only */
[[eosio::on_notify("eosio.token::transfer")]] 
void on_payment (name from, name to, asset quantity, string memo) {
  ...
}

/* generic handler catching transfers in every token contract */
[[eosio::on_notify("*::transfer")]] 
void on_transfer (name from, name to, asset quantity, string memo) {
  ...
}
```

In both cases, the default dispatcher will look for a matching handler and try to deserialize the notification arguments accordingly. See also the chapter on [smart-contract-security.md](../design-guidelines/smart-contract-security.md "mention"), outlining potential risks in such handlers. &#x20;

## Calling an action

If an RPC client is packing the transaction, the only information about its arguments is the ABI associated with the contract.&#x20;

But, if we are calling some action from within our contract using  `eosio::action::send()`, _we don't have the ABI_. We can only assume the arguments and their types by looking up the ABI or by reading the source of the called contract, if it's available. If the called contract changes the definition of its action, the calling contract code would have to be updated before it can call that action again.

## Transaction as seen from within an action

Once our smart contract gets control during a transaction execution, it may derive certain information about what was going on before our action call but not all of it.&#x20;

As explained in [life-cycle-of-a-transaction.md](../how-an-antelope-blockchain-works/life-cycle-of-a-transaction.md "mention"), a transaction execution consists of a sequence of WASM virtual machines spawning, and your code cannot determine where in this sequence you are at this specific moment. Your action handler may be called from the top-level list of actions or be a consequence of some other action execution.&#x20;

What we do know is the content of the initial transaction as it was submitted to the blockchain. There is an intrinsic method that copies transaction bytes into a provided buffer. Also, the  CDT provides the transaction and action classes which will help you decode and deserialize the transaction content if needed. But this will only give you the contracts and actions that were called initially plus a list of accounts which authorized the execution.
