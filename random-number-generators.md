# Random number generators

Many blockchain applications require a reliable source of random numbers. These numbers are used for all kinds of random reward distributions in games, online casinos, and NFT sales.&#x20;

Many different projects tried to build on-chain RNG, and many have lost money doing that. The general rule of thumb, DO NOT TRY BUILDING AN ON-CHAIN RNG.

The most primitive ones tried to use the transaction ID as a source of randomness. It was quickly realized that one can craft a transaction with a desired property in its ID: adding a random garbage input, one can try as many times as needed to achieve the desired output before broadcasting a transaction.

Then, there was a game that used the ID of a block that was generated 10 blocks after a given transaction. Then someone found a way to fill the queue of pending transactions so much that the blockchain produced a sequence of empty blocks. The block ID of the next block is predictable if it is empty. So, the intruder emptied the bank within minutes.

In general, any outcome of user inputs on the blockchain can be made predictable, as the blockchain is deterministic.

But, we still want our RNG to be verifiable. We don't want to rely on some third party that is free to inject any numbers at will.

The asymmetric cryptography comes to help: if we know a public key, we can verify a signature made by that key, but there's no way to predict the signature bits without knowing the private key.&#x20;

An Antelope smart contract can verify a _secp256k1_ or _secp256r1_ signature. So, a back-end oracle may sign the user's input with its private key and send the signature to the smart contract. The downside of this method is that multiple signatures may match the same input. Both algorithms include a random factor that is used to produce a valid signature. So, in theory, the oracle may still manipulate the output and pick the signature that generates a desired outcome.

The RSA (Rivest–Shamir–Adleman) cryptographic algorithm does not use a random factor for signing, so the same inputs and private keys will always produce the same signature. As a result, the RNG that is built on top of RSA is always provable and it eliminates the possibility for cheating. One downside is that the standard Antelope Leap software does not have the built-in RSA signature verification, so the  RNG smart contract would have to do the heavy lifting of RSA calculations. WAX blockchain, however, added an intrinsic to the Leap node software  to perform the signature validation natively. This reduces the CPU workload significantly.
