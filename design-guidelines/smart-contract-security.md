# Smart contract security

This chapter is a copy of my older [blog post](https://cc32d9.medium.com/eosio-contract-security-cookbook-20210527-69797efe9c96).

## Check the token contract <a href="#3403" id="3403"></a>

Older CDT versions did not offer a nice wrapper for `apply()` handler, so the contract authors needed to pay more attention on notifications. Current CDT screens these mechanics by predefined macros. Still, you can create an universal notification receiver, like:

```clike
[[eosio::on_notify("*::transfer")]] 
  void on_payment (name from, name to, asset quantity, string memo) {...}
```

this will process all `transfer` action notifications where your contract is either To, or From. The notification can come from a standard token contract, or from a specially crafted contract that simulates the token behavior. So, before assuming the payment as valid, always check that the token contract is what you expect. The contract account is retrieved by calling `get_first_receiver()`.

There are token contracts handing multiple currencies, so it’s also important to validate the symbol in `quantity`.



## Check that `to == _self` <a href="#6243" id="6243"></a>

In transfer notification, a common logical error by many contract authors was that if `from` is not me, it must be a payment for me. Well, it’s wrong.

Let’s say Alice and Bob have a smart contract that has the transfer notification handler that calls `require_recipient(Chris)`. Alice sends a token to Bob, and Chris receives two transfer notifications with a valid token contract. But he is neither sender nor receiver. If Chris’s contract doesn’t check who the `to` is, it accepts the payment as incoming. If you run a game like casino or a web shop, you treat it as real money and send back the real value. And Alice and Bob can send the token back and forth until you’re drained. This has happened multiple times, especially in early EOS days.

So, your notification handler should look like this:

```
[[eosio::on_notify("*::transfer")]] 
  void on_payment (name from, name to, asset quantity, string memo) {
    if(to == _self) { 
      name tkcontract = get_first_receiver(); 
      // look up tkcontract in your list of accepted currencies 
      // verify the token symbol  
      // process the payment  
      }
  }
```

## Do not query tables from speculative nodes <a href="#e182" id="e182"></a>

This attack happened in the end of 2018 and many casinos on EOS were drained.

`nodeos` has several options in how to handle `get_table_rows` calls. By default, the node returns the table rows which may be altered by speculative transactions which are in transit, but have not been included in a block yet.

So, the attackers found (or maybe just tried) which casinos were quering speculative nodes, and injected their bets in such a way that they failed before being included in a block. The casino was reading from speculative state, and assuming that the bets were made, and sending back the wins in real money. The attacker didn’t even spend any token because the bet was placed by a contract that fails after a certain short time period.

So, the API node needs to be configured to either use `read-mode=head`, or to disable speculative transactions coming in. And best of all, if the contract is subject to such a vulnerability, the dapp needs their own API nodes under their own control. Also the game contracts need to verify that the bet is valid before sending back the rewards.

## Assume that an intruder injects inline actions between yours <a href="#afa1" id="afa1"></a>

This is [what happened in EOSX Vault](https://cmichel.io/eos-vault-sx-hack/). The contract assumed a certain sequence of action and notification calls, and the intruder made a smart contract that reacts on notifications and injects inline actions between the vault’s.

So, the rule of thumb is, never assume that your contract is called in a certain sequence. There will be a moment when it is not completely true and there’s an unexpected call in the middle of your workflow. Build your workflows with this in mind.

## Asset can be negative <a href="#87d5" id="87d5"></a>

Developers sometimes forget that the `asset` type allows negative values. Whenever a user submits an amount of currency to your contract, check that it is positive.

Also worth noting that the 64-bit signed amount may overflow, and the attacker may exploit that as well.

## Other checks <a href="#dfff" id="dfff"></a>

The rest is just usual precautions when designing a smart contract:

* check authorization of the caller
* memory control: either allocate RAM from the caller’s pool, or make the calls cost something, to mitigate a DOS attack on your RAM pool.
* sanity checks wherever possible. Do not assume that your contract works perfectly. Always check the values returned from contract tables, even if you’re sure they cannot be wrong. I do the following quite often:\
  `auto ciitr = ci.find(w.template_id);`\
  `check(ciitr != ci.end(), "Exception 3"); // this must never happen`
* always let a fresh eye look at your code. The human brain assumes too much and it is blind to small mistakes in its own creation. Let someone else look through it, or order a proper security audit. Let others run tests on your code.
