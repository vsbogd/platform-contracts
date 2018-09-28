# Overview

Below proposal describes changes in APIs of SingularityNet components. Main
goal is allowing step by step migration of the SingularityNet from Jobs to
MultiPartyEscrow contracts. It means that for some time it will be possible to
use both payment ways.

# IPFS metadata changes

Current API:

- metadata
  - description
  - modelURI

Proposed API:

- metadata
  - version - used to track format changes
  - (old) description - service description
  - (old) modelURI - IPFS URI to the .tar archive of protobuf service specification
  - reference_price - price per operation;
  (!) can be replaced by invoices from server;
  (?) should it be different for each group?
  - group[] - group is the number of endpoints which shares same payment channel; grouping strategy is defined by service provider; for example service provider can use region name as group id
    - group_id - unique id of the group
    - payment_address - Ethereum address to recieve payments
  - endpoint[] - address in the off-chain network to provide a service
    - endpoint_uri - http://127.0.0.1:1234 - unique endpoint identifier
    - group_id

# Agent contract API changes:

## Introducing replicas and Multi Party Escrow (MPE)

As all payment related logic will be moved to the MPE contract and some
metadata will be moved to IPFS, most of the fields and methods will be
obsolete.

Fields to be removed:
- token - moved to MPE
- createdJobs - moved to MPE
- currentPrice - moved to IPFS metadata
- endpoint - moved to IPFS metadata

Methods to be removed:
- setPrice - replaced by Agent.setMetadataURI
- setEndpoint - same
- createJob - replaced by MPE.open_channel()
- fundJob - replaced by MPE.deposit()
- validateJobInvocation - replaced by payment validation in snet-daemon
- completeJob - replaced by MPE.channel_claim()

Proposed Agent API:

Fields:
- state - service state
- owner - address which can modify metadata
- metadataURI - IPFS metadata JSON URI

Methods:
- enable - enable service
- disable - disable service
- setMetadataURI - update metadata

## Replacing Agent by Registry plus metadata URI

We can move further and replace Agent contract by keeping metadataURI and service
state in Registry instead of using separate Agent contract.

# Daemon API changes

Sequence diagram of calls during client/daemon interaction:
![Client/daemon interaction sequence diagram][clientDaemonInteractionSequenceDiagram]


## RPC call API

Current API:

- GRPC request metadata:
  - snet-job-address
  - snet-job-signature

Proposed API:

- GRPC request metadata:
  - request-type - if this field is absent it is an old Job based protocol; with
    "mpe" in this field it is a new protocol with MPE contracts.
  - payment-channel-id - id of the payment channel in MPE contract
  - payment-amount - payment amount authorized by client
  - payment-signature - client payment signature

## Blockchain events API

New MPE events doesn't intersect with previous Job one. NewPaymentChannel
events should be polled from Ethereum blockchain and added to the distributed
payment channel cache.

# Development plan

1. Add MultiPartyEscrow contract
2. Add separate snet-cli command ```mpe``` to interact with service using MPE API:
  - publish service
  - add group of replicas
  - add replica
  - deposit tokens to MPE
  - open payment channel
  - close channel after expiration
  - call service using opened channel
3. Implement separate request handler in snet-daemon to handle requests using
   MPE payment channels
   - listen channel related blockchain events
   - get new metadata from requests
   - verify payment authorization
   - claim tokens from payment channel
   - share state within group of replicas
4. Implement dApp using MPE contracts
5. Remove legacy API

[clientDaemonInteractionSequenceDiagram]: https://www.plantuml.com/plantuml/svg/bLHhSw985FtEhw1cFwofbGI3egOgkmEeu1jHVAMcgBPjfUNH28seltv9HCUOcMJt2mNTvvvxxk7U-oThKnf4JmyFIPBS1oxmAKLxUW-9z_3FwzlpejCg61li2JArfNZkz66jZilrWb6WW_veDbNZeZav21DTYJK9EARsEBRfbglgjL4KhPjxKJOEN6CUQmrPL0BR4MVRedJi5plM63H61kYhOkCe3USLR-cKQlqw97WwWEQ-D5kjxSuG6tPxvkqB_J4jj8tMNh262dNgUqkVk-sosPiD2nDrqrG8Fsw4NCpn0vFKcxRUqCiUd9EWL37MlAdndB0dLbkigV5nwFTvPTpnYmufQJEduy6HqL_mL9BzLO5JkKgXFTChExpSBAHcs_6492BhiYrvAwyasejXYn0XgCmM6crMjk74t2xtckKvwc9dJ6iBXECfjvCQLKaZMrmCEMcdLwEqc4c4uzIwqjACUMpnfPMiM0alMtRB6HSdngW_LdgRBcnLBVrnYttz43RaGdXeYbBmE8NoSevwoVWe7sSbtW204zeWD9tONQd-aOnLiTabmEVcPLayE9ohQ2EffMwEUanmLLvjpMrbkwahDSK0eu6d7XO52EibqHr8crrFQ9XhZPJ5uDZiLES3kze6xP9LD3RqwAvcOS_rXMfarI0A7w7FDnDH7A66eMCuR26jhzMKRYaPWTe22sB2ALEtwWcJOPighha1Gcjbo5tjqBK6KknKd7ezcWx66nKPk1BibON5V8ye26dikGn3CNKH-wsBeWYiKSHox11jO-H3nDOmM8V0-yOmwv340Rj40OamPHGN8v-otCiV8Lh--9ljwdNswNoRsm4tHdVCws6AM8S8-Ib6KfifnYCxP57Y83_AmJUM_6jqdhqD0eSBxmAa4m9Hz78A-C7-zNo2CyXVClVtzvaw4Y2VXJRmVUJ-Hk8hnCeWkGY5EmnHbZlFeds0Gq0nyVDiY08NGs2zCLzadQXg07d4JzbwAD71mKlpA1bbHdrtbJuh9JjuOgFOUybW5bx-O9zP_gQEyDN-YBAKi9d4_q6NUXE4duS49yrfvWYlVIlxpFraoT-otdsmy-919ZQt11JapzHtJ7R_C_ib0De8EfUaBeZe-OTOQKzIz1kIKnCDJtxSvBrZpkTNq2k5KK3y2ENIbzYzehsov0Fc_H-1nASXW3IL5b8CN3R1r5w6807kbSZhHZkN15s0lLnMKPvT0JVwhrtEFP-Tk3zr-wTHMQUS6k3BC1ybbvRDfl3zDAHRu1tBBwVZPYukRlvYBZ83RutzqiSpuCDI-TG3xhmWljn1RxldVUadZSptz9dknty1 "Client/daemon interaction diagram"
