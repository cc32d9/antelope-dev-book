# Customizing the Antelope software

## Customizing the `eosio.token` contract

In the past five years, many projects have tried adding their own functionality into the default token contract. In general, it is advised not to do so by all means, if possible, for a number of reasons:

* The default token contract has been validated and audited by many pairs of eyes. It has never been compromised, and can be considered to be completely secure. The code is also very simple, and that adds up to the level of security. Adding any modifications may lead to unforeseen security problems, which could lead to a significant financial loss.
* Whenever someone needs to build a financial report, one assumes that the balances are strictly updated by transfer actions only. If the modified token contract alters the token balances by some internal logic without reflecting the changes in a transfer action, any kind of financial auditing and reporting becomes nontrivial.

If a project has to implement a custom logic in the token (such as a deflationary token that burns a part of the asset on every transfer), the contract needs to issue corresponding inline transfer actions, so that the blockchain history can reflect the balance changes accurately.

## Customizing the public key format

EOSIO/Antelope software uses the legacy format for public keys by default, like `EOS77qVt59tgxMos8Ru5Wqz7yLUwAT95v2pFf7CYWpS3CW7quE1we`. It also supports a neutral notation, like `PUB_K1_77qVt59tgxMos8Ru5Wqz7yLUwAT95v2pFf7CYWpS3CW7q2wB35` (which results in the same 256-bit public key in binary form). The neutral form supports also secp256r1 keys, using `PUB_R1_xxx` notation. &#x20;

Several projects have attempted to distantiate themselves from the EOS blockchain and introduced their own public key notation, basically replacing the "EOS" prefix with their own. This has lead to a number of problems, and made it expensive to maintain such blockchains:

* Many open-source software projects became incompatible:  wallets, informational websites, back-end tools, and also the middleware libraries. Such a blockchain would have to maintain their own copy of libraries and tools, which adds a significant effort.
* Most block producers operate on multiple blockchains and share their hardware among them. It adds an additional effort for the server operator, as they have to maintain a separate set of binaries. Also, node configuration and scripts would have to be adapted to the new key format.

Instead of modifying the legacy key format, it is recommended to use the neutral form and avoid such customization if possible.

## Customizing the node software

Multiple blockchain projects have added their customization to the node software. In some cases, this customization is so deep that it takes a significant effort to keep up with the upstream software.&#x20;

### Adding new intrinsics

In some cases, the standard set of intrinsic functions in `nodeos` is insufficient for the needs of a blockchain project, and new functions have to be added.&#x20;

The architecture of `nodeos`, although built in a modular fashion, does not allow intrinsics to be defined in plugins. All of them are implemented within the `chain` library. Also, the library maintains a linear index of intrinsic functions, and it strictly recommended not  to reorder them. It means that new intrinsics should only be added at the bottom of the list. If a project added its own intrinsic, and the upstream Leap software has added more, the new additions should be added after the custom one. If a new software release reorders the intrinsics, the OC compiler cache needs to be cleared manually, so such an upgrade would cause a lot of confusion.

If the changes are done in a granular and isolated fashion, synchronizing with the upstream software updates is usually an easy task.

### Changing the node behavior

Some projects require a modification in default node behavior when it comes to creating accounts or registering producers. The general rule of thumb is to keep the node software as neutral as possible, and perform all the changes in the system contracts. Changes in smart contracts are much cheaper to maintain, and they do not require a global network upgrade.

If the changes in nodeos are still necessary, it is recommended to implement them as features that need an activation on the chain. This will ensure that the unmodified `nodeos` ("vanilla nodeos") would not be able to accept the blocks and synchronize with the blockchain.

Also, such changes need to be made as subtly as possible, modifying as little as possible in the original sources. This will ensure that updates from the upstream software are merged easily without a major effort.

### Renaming the software components

The main software components of Antelope software still have "EOS" in their names, like `nodeos`, `cleos`, `keosd`.  It is tempting for a project to rename them in order to keep the brand distinct from EOS.

The drawbacks of such renaming are more significant than benefits: the legacy names are used in many tutorials and automation scripts. Also, system administrators got already  used to the legacy names, and this change would only increase the struggle of network maintainers.

In addition to that, such a rebranding needs to respect the open-source licenses and copyright of the original software. Violating the licenses may lead to unexpected copyright violation claims and lawsuits.&#x20;
