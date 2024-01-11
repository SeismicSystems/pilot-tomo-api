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
### Authenticating with an ETH wallet
- 

### Sharing an Intent to Signal
### Fetching matches

## Interfacing with your target chain
### Broadcasting a signal
### Verifying signals
