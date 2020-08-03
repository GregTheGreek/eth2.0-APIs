# Validator Flow

Detail explanation how validator should utilize this API to perform his regular BeaconChain duties.


### Block Proposing

On start of every epoch, validator should [fetch proposer duties](#/Validator/getProposerDuties).
Result is array of objects, each containing proposer pubkey and slot at which he is suppose to propose.

If proposing block, then at immediate start of slot:
1. [Ask Beacon Node for BeaconBlock object](#/Validator/produceBlock)
2. Sign block
3. [Submit SignedBeaconBlock](#/ValidatorRequiredApi/publishBlock) (BeaconBlock + signature)

Monitor chain block reorganization events (TBD) as they could change block proposers. 
If reorg is detected, ask for new proposer duties and proceed from 1.

### Attestation

On start of every epoch, validator should ask for attester duties for epoch + 1.
Result are array of objects with validator, his committee and attestation slot.

Attesting:
1. Upon receiving duty, have beacon node prepare committee subnet
    - [Check if aggregator by computing `slot_signature`](https://github.com/ethereum/eth2.0-specs/blob/v0.12.2/specs/phase0/validator.md#attestation-aggregation)
    - [Ask beacon node to prepare your subnet](#/ValidatorRequiredApi/prepareBeaconCommitteeSubnet)
      -- Note, validator client only needs to submit one call to
      `prepareBeaconCommitteeSubnet` per committee/slot it's validators have
      been assigned to. If any validators are aggregators, be sure to use the aggregator's
      `slot_signature` to properly signal aggregation to the beacon node
2. Wait for new BeaconBlock for the assigned slot (either stream updates or poll)
    - Max wait: `SECONDS_PER_SLOT / 3 * 1000` into the assigned slot
2. [Fetch AttestationData](#/ValidatorRequiredApi/produceAttestationData)
4. [Submit Attestation](#/ValidatorRequiredApi/submitPoolAttestations) (AttestationData + aggregation bits)
    - Aggregation bits are `Bitlist` with length of committee (received in AttesterDuty)
    with bit on position `validator_committee_index` (see AttesterDuty) set to true
5. If aggregator:
    - Wait for `SECONDS_PER_SLOT * 2 / 3` into the assigned slot
    - [Fetch aggregated Attestation](#/ValidatorRequiredApi/getAggregatedAttestation) from Beacon Node you've subscribed to your subnet
    - [Publish SignedAggregateAndProof](#/ValidatorRequiredApi/publishAggregateAndProof)

Monitor chain block reorganization events (TBD) as they could change attesters and aggregators. 
If reorg is detected, ask for new attester duties and proceed from 1..