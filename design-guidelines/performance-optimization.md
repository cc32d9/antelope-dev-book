# Performance optimization

## Avoiding overhead in `check()`

`check()` is a function defined by the CDT, and it takes a boolean and a string. If the boolean is false, the whole transaction is aborted, and the string is output to the user as an error message.

It's tempting to concatenate strings and insert dynamic values, like this:

```
check(accounts_itr != _accounts.end(), 
      "Account " + account_name.to_string() + " is unknown");      
```

But `check.hpp` in CDT is defining `check()` as an inline function with two or three arguments, so it evaluates the string every time before checking the predicate. So, this concatenation in the example above and name-to-string conversion is happening every time the check is called. If you have many such check-ups, or if you're verifying something in a loop, you automatically get an overhead of unnessessary string conversion. The refactoring as below will reduce the CPU effort for the times when the predicate evaluates to true (and you really don't care how much CPU time is needed to compose the error message in case of a failure):

```
if( accounts_itr == _accounts.end() ) {
  check(false, "Account " + account_name.to_string() + " is unknown");
}
```

## Accessing table rows

As described in [data-design.md](data-design.md "mention"), every time you access a table row, two operations take place: first, nodeos is finding the row in its internal multi-index structures (logarithmic complexity on the  number of all the table rows in state memory). If you are using a secondary index, this operation is performed twice (finding the entry in the index, then finding the row by primary key). Then, the row contents are deserialized from a sequence of bytes into a structure in VM memory (linear complexity on the size of the row).

So, if you have a dilemma between having longer rows or having to retrieve the rows more frequently,  the best way is to test and profile it on a running blockchain. Keep in mind that the time to find a row in global memory is growing with the growth of the total blockchain state, so the best way is to perform a few tests on a production blockchain.

## Powers of ten

If a contract needs to calculate powers of 10 (for example, to convert a raw integer asset value into a floating-point number according to the asset precision), sometimes developers utilize the standard C++ `pow()` function. It will work, but one may notice a high CPU cost for such an operation.

A more efficient way would be to have an array of pre-calculated powers of 10 and to perform a simple lookup for the desired number. In fact, CDT delivers [powers.hpp ](https://github.com/AntelopeIO/cdt/blob/main/libraries/eosiolib/core/eosio/powers.hpp)which is doing exactly that for any given base.

## Delegating jobs to external workers

A contract may need to perform CPU-intensive calculations or data manipulation. If it loads the end user's transaction with this job, it may lead to a degraded user experience: the users will need to have a higher CPU allowance, and they will see more frequent errors because they don't have enough CPU respurce.

One of possible approaches is to queue such jobs for later processing, and let anyone trigger it. Then a backend oracle script would trigger the processig as frequently as needed.&#x20;

It is also possible to keep a counter of executed jobs for each worker, and define a reward for them. If the reward is high enough, third party users would be motivated to set up the backend scripts and trigger the processing jobs concurrently. This approach may lead to an interesting "mining" economy.

It is important for the smart contract to reject a transaction which has nothing to process. Otherwise the blockchain history would be polluted with useless transactions, which would lead to higher maintenance costs and degraded user experience.
