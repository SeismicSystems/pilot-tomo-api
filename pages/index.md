---
title: Building on-chain swipes with Tomo <> Seismic
description: API docs for the Tomo <> Seismic pilot integration.
---

# Building on-chain swipes with Tomo <> Seismic

## An overview of this integration

### The what and why for on-chain swiping
Tomo's implementing fully on-chain swipes to leverage the composability property of blockchains. The team wants anyone to be able to run experiments on top of their swipe social graph. Think of the alternate world where Tinder's graph of swipes was composable. Then Hinge, Bumble, and every other dating app could've leveraged and added to its user base. 

That's the boring example though. The real value add is with the experiments that may never exist if they needed to bootstrap their own networks. Imagine "Tinder (Date Edition)", an app that organizes date parties around clusters of likes. Imagine "Tinder (Second Chance)", an app that connects users who should give potential matches second chances. These are only two ideas out of an interesting design space of experimental social interactions that could be tested with composable swipe graphs. 

The predominant issue with this plan: on-chain swipes are public if done in the default way. This is a non-starter. Users aren't comfortable with romantic interests being public. Even with the pseudonymous nature of wallets, it still isn't acceptable for user A to know that user B has 100 other concurrent matches. This is why Tomo is partnering with Seismic to shield their on-chain swipes.

### How Seismic shields Tomo's Swipe Graph
Shielded swipes fit well within Seismic's standard cryptosystem. Here's the general idea. The default method is to directly broadcast the swipe from `userA` to `userB` to the chain. Instead of doing this, push a hiding commitment to the swipe. 

Observers can see that `userA` just did some protocol action, but cannot see the content of the swipe, nor who the swipe was directed at. The pre-images of these hiding commitments are stored in Seismic's auxiliary sequencer running in a secure enclave, which is there so even Seismic operators cannot see the sensitive social graph without a sophisticated exploit. 

Seismic's job is to reveal symmetric likes. If there exists a commitment on-chain that reflects `userA` liking `userB`, and vice versa, then Seismic tells both users about the match. With any other condition, Seismic will not reveal the contents of swipes.

A sample user sequence may proceed as follows:
1. Alice likes Bob. She swipes right on her phone. 
2. Alice's Tomo client alerts Seismic's sequencer of `(Alice, Bob, true)`.
3. Seismic's sequencer signs the hiding commitment `H(Alice, Bob, true)` and sends it back to Alice. 
4. Alice can then broadcast `H(Alice, Bob, true)` and Seismic's signature to the chain. Notice that this uses the same mechanism as data availability solutions to ensure pre-images are logged before action happens on-chain. 
5. Bob at some future date is presented with Alice's card and likes her. He goes through steps 1 - 4.
6. After Alice's and Bob's transactions are finalized, both are entitled to seeing the match.

It's a specific workflow that has little room for deviation without disrupting the security model. Note that this integration only covers the backend for swipes. Recommending cards, rendering the swipes, and chatting with matches is not covered.

## Interfacing with Seismic

### Retrieving the nonce for authentication
#### >> Description
Tomo's client interacts with Seismic's sequencer through HTTP requests. The authentication procedure is the same as Ethereum's: sign the `keccak` hash of the request, nonce, and domain separator with an ETH private key. 

We use the standard deterministic ECDSA signature with the following type:
```
type Signature = {
    r: string,
    s: string,
    v: string
};
```

We allow anyone to get their incrementing nonce through this endpoint. The nonce counts the number of transactions between this wallet and Seismic, not the Ethereum network.

#### >> Network Calls

**Request**

```
GET /tx_count
```

**Request Body**

```
{
    address: string
}
```

**Response**

```
{
    txCount: number
}
```

### Sharing a claimed swipe

#### >> Description
Tomo's client communicates with Seismic to claim that it is about to register a swipe. It's a "claim" because the swipe isn't actually registered until it's executed on-chain. 

Signing the claim can be done using the below reference snippet with any library providing EIP-721 support. We base all sample code in this doc off of [viem](https://viem.sh/)'s API.
```
const requestSig = await walletClient.signTypedData({
    walletClient.getAddress(),
    {
        name: 'Tomo Swipe Authentication',
        version: env['VERSION'],
        chainId: env['CHAIN_ID'],
        verifyingContract: env['CONTRACT_ADDR'],
    },
    {
        Swipe: [
            { name: 'recipient', type: 'address' },
            { name: 'positive', type: 'bool' },
            { name: 'blind', type: 'uint256' },
        ],
        Claim: [
            { name: 'nonce', type: 'uint256' },
            { name: 'body', type: 'Swipe' },
        ]
    },
    primaryType: 'Claim',
    message: {
        nonce: 3,
        body: {
            recipient: 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2,
            positive: true,
            blind: 40240711473155685292128167834219056567299322304092574000529763031142493119900,
        }
    }
});
```

Seismic acknowledges the claim by signing the commitment to the swipe. We include the snippet below for completeness, even though it's code that exists only on Seismic's side. Tomo should, however, verify that the commitments signed by Seismic correspond to their claimed swipe.
```
const swipeCommitment = keccak(
  abi.encodeParameters(['address', 'bool', 'uint256'], 
  [recipient, positive, blind])
);

const responseSig = await walletClient.signTypedData({
    walletClient.getAddress(),
    {
        name: 'Tomo Swipe Acknowledgement',
        version: env['VERSION'],
        chainId: env['CHAIN_ID'],
        verifyingContract: env['CONTRACT_ADDR'],
    },
    {
        Commitment: [
            { name: "value", type: 'uint256' }
        ]
    },
    primaryType: 'Commitment',
    message: {
        value: swipeCommitment
    }
});
```

#### >> Network Calls

**Request**

```
POST /claim
```

**Request Body**

```
{
    claim: Claim,
    signature: Signature
}
```

**Response**

```
{
    commitment: Commitment,
    signature: Signature
}
```

### Fetching matches

#### >> Description

Tomo's client uses Seismic to fetch all matches that have been finalized on-chain. Note: claimed matches that aren't confirmed will **not** be included.

The fetch request must include an index to specify which match number to start at when sorted by block height of chain confirmation. Seismic returns at most 100 matches at a time for one user. If Tomo's client is more than 100 behind for a user, it can sync through repeated requests. 

As with claimed swipes, we provide a sample code snippet for signing a request to fetch matches:
```
const requestSig = await walletClient.signTypedData({
    walletClient.getAddress(),
    {
        name: 'Tomo Swipe Matches',
        version: env['VERSION'],
        chainId: env['CHAIN_ID'],
        verifyingContract: env['CONTRACT_ADDR'],
    },
    {
        Fetch: [
            { name: 'nonce', type: 'uint256' },
            { name: 'startIndex', type: 'uint256' }
        ]
    },
    primaryType: 'Fetch',
    message: {
        nonce: 4,
        startIndex: 170
    }
});
```

#### >> Network Calls

**Request**

```
GET /matches
```

**Request Body**

```
{
    fetch: Fetch,
    signature: Signature
}
```

**Response**

```
{
    matches: Swipe[]
}
```

## Interfacing with your target chain

### Broadcasting a signal
The swipe is registered directly from the user to the chain. Users must send the commitment, along with Seismic's signature of acknowledgement. 
```
swipe(uint256 swipeCommitment, ECDSASignature signature);
```

### Verifying swipes 
Tomo clients should verify that any pre-images sent by Seismic are reflected by the chain. It's a necessary step to uphold our security model.

The contract maintaines a public mapping of `swipeCommitment` to a `bool` that reflects whether it's been registered. The Tomo client can use this to verify the pre-images. 
```
swipeContract.methods.swipes(swipeCommitment).call()
    .then(isPresent => {
        if (isPresent) {
            console.log("Swipe verified.");
        }
        else {
            console.error("Does not match a swipe committed to in contract.");
        }
    })
    .catch(error => {
        console.error("Error reading contract swipes:", error);
    });
```
