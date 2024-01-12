# Tomo's building on-chain swiping w/ Seismic

## An overview of this integration

### The what and why for on-chain swiping
Tomo wants access to the composability provided by having fully on-chain swipes. The team wants anyone to be able to run experiments on top of their application without needing to boostrap their own social graphs. To make this concrete, think of the alternate world where Tinder's graph of swipes was composable. Then Hinge, Bumble, and every other dating app could've leveraged and added to its user base. 

That's the boring example though. The real value add is with the experiments that may never exist if they needed to bootstrap. Imagine "Tinder (Date Edition)", an app that organizes date parties around clusters of likes. Imagine "Tinder (Second Chance)", an app that connects users who previously disliked each other but should give it another shot. These are only two ideas out of an interesting design space of experimental social interactions that could be tested with composable swipe graphs. 

The dominant issue with this plan: on-chain swipes are public if done in the default way. Users aren't comfortable with romantic interests being public. Even with the pseudonymous nature of wallets, it still isn't acceptable for user A to know that user B has 100 other concurrent matches. This is why Tomo is partnering with Seismic to shield on-chain swipes.

### How Seismic shields Tomo's Swipe Graph
Shielded swipes fit well within Seismic's standard cryptosystem. Here's the general idea. Instead of directly broadcasting the swipe from `userA` to `userB` to the chain, push a hiding commitment. Observers can see that `userA` just did some protocol action, but cannot see the content of the swipe, nor who the swipe was directed at. 

The pre-images of these hiding commitments are stored in Seismic's auxiliary sequencer running in a secure enclave, which is there so even Seismic operators cannot see the sensitive social graph without a sophisticated exploit. Seismic's job is to reveal symmetric likes. If there exists a commitment on-chain that reflects `userA` liking `userB`, and vice versa, then Seismic tells both users about the match. With any other condition, Seismic will not reveal the contents of swipes.

A sample user sequence may proceed as follows:
1. Alice likes Bob. She swipes right on her phone. 
2. Alice's client alerts Seismic's sequencer of `(Alice, Bob, true)`.
3. Seismic's sequencer signs the hiding commitment `H(Alice, Bob, true)` and sends it back to Alice. 
4. Alice can then broadcast `H(Alice, Bob, true)` and Seismic's signature to the chain. Notice that this is the same mechanism as with data availability to ensure pre-images are logged before action happens on-chain. 
5. Bob at some future date is presented with Alice's card and likes her. He goes through steps 1 - 4.
6. After both Alice's and Bob's transactions are finalized, both are entitled to seeing that they've matched.

It's a specific workflow that has little room for deviation without disrupting the security model. Note that this integration only covers the backend for swipes. Recommending cards, rendering the swipes, and chatting with matches is not included.

## Interfacing with Seismic

### Nonce for authenticating with an ETH wallet
- authenticating with seismic the same as with ethereum network
- sign the keccak hash of your request + a nonce using your ETH privkey, with
  an incrementing nonce to prevent replay attacks
using ECDSA signatures
```
type Signature = {
    r: string,
    s: string,
    v: string
};
```

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

- address A claiming they will register a swipe with address B
- claim because they may not actually do this on-chain, in which case it's
  never true
- share claim with Seismic before actually swiping
- authentication by signing the object with claim and nonce in it, not sig
- signs the commitment to the swipe to confirm reception and sends it back
- client can verify that the commitment corresponds to their intended swipe

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
        ClaimBody: [
            { name: 'addressA', type: 'address' },
            { name: 'addressB', type: 'address' },
            { name: 'positive', type: 'bool' },
            { name: 'blind', type: 'uint256' },
        ],
        Claim: [
            { name: 'nonce', type: 'uint256' },
            { name: 'body', type: 'ClaimBody' },
        ]
    },
    primaryType: 'Claim',
    message: {
        nonce: 3,
        body: {
            addressA: 0x00000000219ab540356cbb839cbe05303d7705fa,
            addressB: 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2,
            positive: true,
            blind: 40240711473155685292128167834219056567299322304092574000529763031142493119900,
        }
    }
});
```

```
const swipeCommitment = keccak(
  abi.encodeParameters(['string', 'string', 'bool', 'string'], 
  [addressA, addressB, positive, blind])
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

- get all matches that have been finalized on-chain, note this means claimed
  matches that aren't confirmed will not be included
- must specify an index for which match number to start at, only returns 100
  matches at a time
- 

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
        ]
    },
    primaryType: 'Fetch',
    message: {
        nonce: 4,
    }
});
```

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
    matches: ClaimBody[]
}
```

## Interfacing with your target chain

### Broadcasting a signal
- send the commitment to the chain, along with the signature of Seismic 
  acknowledging the pre-image
```
swipe(uint256 swipeCommitment, ECDSASignature signature)
```

### Verifying swipes 
- need to do for security model
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
