# Tomo's building on-chain swiping w/ Seismic

## An overview of this integration

### The what and why for on-chain swiping

- many blockchain properties useful for different applications
- for Tomo, the main motivation is composability
- if all likes / dislikes on-chain, can launch experiments on top w/o
  needing to bootstrap a social graph
- if tinder social graph composable, can build hinge / bumble / etc
- but that's the boring example
- now can run experiments on ideas that are much harder to bootstrap, eg
  Tinder (Date Edition), an app where you can organize date parties and around
  clusters of likes
- problem with on-chain swipes, all public
- not good for romantic interests to be public
- even in the pseudonymous world, isn't good for user A to know that user B has
  100 other concurrent matches
- so partnering with Seismic to shield (hide) swipes between
  wallets

### How Seismic shields Tomo's Swipe Graph

- fits well within Seismic's standard crypto system
- instead of directly broadcasting (userA, userB, like_bool) to chain, push a
  hiding commitment
- the pre-image of these commitments are stored in Seismic's auxiliary sequencer
  running in a secure enclave
- then if symmetric like, Seismic reveals
- sample flow
  - Alice likes bob, swipes right
  - Alice's client alerts seismic of (Alice, Bob, true), Seismic signs, a-la
    data availbility style
  - Alice sends H(Alice, Bob, true) to contract
  - Bob, at some future date does the same for (Bob, Alice, true)
  - after this, next time Alice and Bob request for their matches,
- doc only for swipes, recommending cards up to Tomo

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
