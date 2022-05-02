---
title: Simple EIP712 Verifier
date: 2022-05-01T21:40:46-05:00
draft: true
---

#### Repo link at the bottom

Quoting from the official proposal [here](https://eips.ethereum.org/EIPS/eip-712)

> Signing data is a solved problem if all we care about are bytestrings. Unfortunately in the real world we care about complex meaningful messages. Hashing structured data is non-trivial and errors result in loss of the security properties of the system.

> As such, the adage “don’t roll your own crypto” applies. Instead, a peer-reviewed well-tested standard method needs to be used. This EIP aims to be that standard.

As always, there's a relevant [xkcd for this](https://xkcd.com/927/).

This post will aim to briefly show how this _standard_ works and how you verify off-chain data in a "safe" manner.

Let's imagine we're in the following scenario: **You have a big application where you give a token to users, users can redeem this token for another type of reward based on a a series of attributes that this token has. Sadly, this token's attributes are not recorded on-chain. ZKSnarks are not a good use-case here.**

- You have a public facing function `giveReward` that receives parameters `i`, `level`, and `reward`, `level` is recorded off-chain.
- `i` is the token ID.
- You need to make sure that `reward` is not modified since it dictates the reward that you will send.

In the current state, this function can be manipulated in such a way that `reward` differs from what was initially calculated off-chain.

```solidity
function giveReward(uint256 i, uint256 level, uint256 reward) external {}
```

**EIP712** is a way to solve this issue, in the following way:

- `giveReward` would receive 2 additional parameters, say `signer` and `signature`.
- `signer` will be the address against which we want to match the creator of `signature`.
- `signature` will be a hash computed off-chain.

```solidity
function giveReward(uint256 i, uint256 level, uint256 reward, address signer, bytes32 signature) external {
    require(isValidSignature(i, level, reward), "invalid signature");
}

function isValidSignature(...params) internal view returns (bool) {
    bytes32 digest = _hashTypedDataV4(keccak245(params));

    return ECDS.recover(digest, signature) == signer
}
```

Basically what we need to do is compute a hash of an user defined struct. This struct usually contains the keys we want to verify, the number of arguments your function accepts depends on the number of fields you want to verify in the hash.

Say that your request from the example above needs to have a expiry date. You would need to accept a new argument `uint256 expireAt`. Verify the hash you received with the arguments you were provided and then use the arguments for the business logic.

This process is, of course, not 100% safe. Would the private key of your signer get compromised, anyone could create fake signatures and have free access to the whole function.

On top of that, the example I've provided has several flaws.

- It does not invalidates signatures (although repeating a hash is very unlikely, it's still a good practice)
- You could have an internal `nonce` calculator for a specific address so you avoid replay attacks.
- Instead of accepting a `signer` on the arguments, since this can be faked by anyone, you should rely on internal state of your contract. You can match the recovered address of the signature has a role in the smart contract, for example. [OZ's AccessControl](https://docs.openzeppelin.com/contracts/2.x/access-control) is a good option for this.

You can even go one step further and combine 2 and 3 with a rolling address queue. Multiple addresses can create signatures but only the address at the head of the queue can execute the next one.

**Treat all the code in this post as non-working examples. A working (vulnerable) repo can be seen [here](https://github.com/aguxez/eip712-verifier)**
