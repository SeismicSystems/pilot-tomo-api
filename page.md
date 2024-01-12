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
- so partnering with Seismic to shield (hide) like / dislike signals between
  wallets

### How Seismic will shield Tomo's Swipe Graph
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

### Sharing a claimed signal
- address A claiming they will signal like / dislike of address B
- claim because they may not actually do this on-chain, in which case it's 
  never true
- share claim with Seismic before actually signalling
- authentication by signing the object with claim and nonce in it, not sig
- signs the commitment to the signal to confirm reception and sends it back
- client can verify that the commitment corresponds to their intended signal

```
const request_sig = await walletClient.signTypedData({
    walletClient.getAddress(),
    {
        name: 'Tomo Swipe Auth',
        version: env['VERSION'],
        chainId: env['CHAIN_ID'],
        verifyingContract: env['CONTRACT_ADDR'],
    },
    {
        ClaimBody: [
            { name: 'addressA', type: 'string' },
            { name: 'addressB', type: 'string' },
            { name: 'positive', type: 'bool' },
            { name: 'blind', type: 'string' },
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
            blind: 115792089237316195423570985008687907853269984665640564039457584007913129639934,
        }
    }
})
```
- request signature is `signTypedData()` on an object with `nonce` and `claim` 
  fields
- commitment is `keccak(abi.encodeParameters(['string', 'string', 'bool', 'string'], [addressA, addressB, positive, blind]))`
- response signature is `signTypedData()` on an object with `commitment` field

**Request**
```
POST /claim
```

**Request Body**
```
{
    claim: Claim,
    signature: {
        r: string,
        s: string,
        v: string
    }
}
```

**Response**
```
{
    commitment: string,
    signature: {
        r: string,
        s: string,
        v: string
    }
}
```

### Fetching matches
- get all matches that have been finalized on-chain, note this means claimed 
  matches that aren't confirmed will not be included
- must specify an index

**Request**
```
GET /matches
```

**Request Body**
```
{
    address: string,
}
```

**Response**
```
{
    commitment: string,  # keccak(addressA, addressB, positive)
    signature: {
        r: string,
        s: string,
        v: string
    }
}
```

## Interfacing with your target chain
### Broadcasting a signal
### Verifying signals
