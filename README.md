# Issue H-1: Aleo prover/network DOS vector due to invalid `split` proofs being free to abuse 

Source: https://github.com/sherlock-audit/2024-05-aleo-judging/issues/34 

## Found by 
dtheo
## Summary

Split transactions that contain only 1 transition (the entire transaction is a single `split` transaction and nothing else) do not require a fee commitment. This allows for a malicious entity to grieve provers by forcing them to attempt and verify invalid proofs with no monetary loss. This can be used by competing provers to DOS their competition and waste CPU cycles that would otherwise be used towards the "mining/proving" the next block. A malicious actor attack multiple provers can significantly slow or stop aleo blocks from being created.

## Vulnerability Detail

Aleo has `fee`, `deploy` and `execute` transaction types. If a `deploy` or an `execute` transaction type is broadcasted with an invalid proof or if the `finalize` fails then their corresponding `fee` transaction will be consumed instead. This is a DOS prevention feature that Aleo uses to prevent the network from having to verify proofs without some amount of collateral fee on the side.

Single split transactions, when called by users directly, are [a special case where the there is no fee commitment required](https://github.com/sherlock-audit/2024-05-aleo/blob/main/snarkVM/synthesizer/src/vm/verify.rs#L225-L248). Aleo attempts to charge the fee directly in the split function by burning it. The issue is that if there is not sufficient funds to pay the fee and the proof fails then their is no backup fee commitment to seize. This opens up the Aleo network to a DOS vector that that is nearly free to abuse. 

An attacker must only pay the initial fees to make a large number of private records with at least 1 microcredit each. As long as they only use each credit once per block , they can reuse the records to to make invalid proofs that will pass all of the transaction prechecks in `check_fee`, `check_transaction`, etc. until the [proof verification is attempted](https://github.com/sherlock-audit/2024-05-aleo/blob/main/snarkVM/synthesizer/process/src/trace/mod.rs#L226-L227) and CPU cycles are wasted.

## Impact

This can be used to DOS Aleo provers by other provers competing to solve the Aleo Coinbase Puzzle (and gain rewards). If a malicious entity attacked all provers at the same time the Aleo network would stop producing blocks during this time. 

## Code Snippet
[snarkVM/synthesizer/src/vm/verify.rs#L225-L248](https://github.com/sherlock-audit/2024-05-aleo/blob/main/snarkVM/synthesizer/src/vm/verify.rs#L225-L248)

[snarkVM/synthesizer/program/src/resources/credits.aleo#L947-L972)](https://github.com/sherlock-audit/2024-05-aleo/blob/main/snarkVM/synthesizer/program/src/resources/credits.aleo#L947-L972)

## Tool used

Manual Review

## Recommendation

Require a fee commitment outside of the split transaction than can be verified quickly in the transaction precheck/speculation flow. 

# Issue H-2: `split` transaction's fixed fees undercharge block stuff DOS attacks 

Source: https://github.com/sherlock-audit/2024-05-aleo-judging/issues/35 

## Found by 
dtheo
## Summary

Split transactions that contain only 1 transition (the entire transaction is a single `split` transaction and nothing else) do not require a fee commitment and instead charge a fixed 10 credit fee. Since the fee is fixed this transaction type can be abused in block stuffing attacks without having to increase the fee during times of congestion. 

## Vulnerability Detail

Similar to Ethereum's EIP-1559 mechanism, Aleo allows for fee increases during times of congestion to prevent DOS due to block stuffing attacks. Unlike public and private `fee` transaction types where the fee is dynamic and provided as inputs, `split` transaction types have a fixed fee that is hardcoded to 10 credits in the `split` function in credits.aleo. This allows for an attacker to use this transaction type to fill blocks to their [max transaction limit](https://github.com/sherlock-audit/2024-05-aleo/blob/main/snarkVM/ledger/block/src/verify.rs#L422-L428) without ever having to increase their fees. This allows for unreasonable cheap block stuffing attacks that can be used to DOS the network during critical time periods to prevent DEX liquidations, collateral adjustments, etc. 

## Impact

The hardcoding of the `split` transactions fee amount and lack of outer fee commitment requirements bypass the Aleo networks dynamic fee mechanism allowing unreasonable cheap block stuff DOS attacks on the network.

## Code Snippet

[snarkVM/synthesizer/src/vm/verify.rs#L225-L248](https://github.com/sherlock-audit/2024-05-aleo/blob/main/snarkVM/synthesizer/src/vm/verify.rs#L225-L248)

[snarkVM/synthesizer/program/src/resources/credits.aleo#L947-L972)](https://github.com/sherlock-audit/2024-05-aleo/blob/main/snarkVM/synthesizer/program/src/resources/credits.aleo#L947-L972)

[snarkVM/ledger/block/src/verify.rs#L422-L428](https://github.com/sherlock-audit/2024-05-aleo/blob/main/snarkVM/ledger/block/src/verify.rs#L422-L428)

## Tool used

Manual Review

## Recommendation

Remove the fixed fee from inside the `split` function in credits.aleo. Require fee commitments for this transaction type that are similar to all other transaction types. 



## Discussion

**evanmarshall**

Somewhat a duplicate of: https://github.com/sherlock-audit/2024-05-aleo-judging/issues/17

A fee is already charged (at the program layer instead of consensus layer). The base fee is about twice was a `transfer_private` requires as a base fee.

# Issue M-1: Validator can block existing delegators from increasing their delegations and new delegators to bond at no cost 

Source: https://github.com/sherlock-audit/2024-05-aleo-judging/issues/23 

## Found by 
dtheo, sammy
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



## Discussion

**evanmarshall**

The ability to `unbond_public` of 0 for a validator is certainly a real issue but I disagree on the impact. The real issue is that we should have some minimum to prevent holding on to more data than necessary in the `unbonding` mapping. As for DOS of delegators by validators, I disagree that this is unintended. Validators can already unbond any of their delegators by design and also prevent any new validators by changing their committee state `.is_open` to false. 

There also is a cost to do this but it's the transaction fee associated with `unbond_public`. In any case, it's an issue we should fix but I would categorize it as `minor`

# Issue M-2: `unbond_public` logic causes issues for some delegators preventing partial withdrawals 

Source: https://github.com/sherlock-audit/2024-05-aleo-judging/issues/31 

## Found by 
dtheo
## Summary

Delegators that share the same withdrawal address as a validator will be unable to make partial withdrawal due to a logic error in `unbond_public`. In some cases this can accidentally trigger the complete unbonding of the valdiator even when the delegator does not intend this. 

## Vulnerability Detail

If a validator ever attempts to delegate to itself or for any reason a delegator shares a withdrawal address with the validator it is bonded to then the logic in `unbond_public` will prevent it from making partial withdrawals. 

Imagine the case:
1. A validator AAA is validating with 100 credits and a withdrawal address of BBB
2. A delegator CCC is bonded to validator AAA with 10M credits and also uses withdrawal address BBB
3. Delegator CCC attempts to unbond ANY amount of credits from the validator using `unbond_public`, lets say 1 microcredit.eg. delegator calls `unbond_public(stakers_addr = CCC, microcredits = 1)`

The following logic to determine if the delegators bond should be entirely removed will always incorrectly report that it should:
```solidity
// Check if the delegator will fall below 10,000 bonded credits.
lt r16 10_000_000_000u64 into r17;

// If the validator is forcing the delegator to unbond OR the delegator will fall below 10,000 bonded credits.
or r11 r17 into r18;

// Determine the amount to unbond: requested amount if >= 10,000 credits, otherwise the full bonded amount.
ternary r18 r5.microcredits r2 into r19;
```

This is due to the fact that the logic to determine if the delegator is being force unbonded by the validator will always incorrectly report that it is. This will cause both the deleagator and the validator to completely unbond and force their credits into the unbonding period (360 blocks). This will happen even if the delegator calls `unbond_public` with 0 microcredits.

## Impact

This can incorrectly force unbonding for delegators and validators.

## Code Snippet

[snarkVM/synthesizer/program/src/resources/credits.aleo#L487-L494)](https://github.com/sherlock-audit/2024-05-aleo/blob/main/snarkVM/synthesizer/program/src/resources/credits.aleo#L487-L494)

I have also created an [annotated pseudo code](https://gist.github.com/infosecual/d6a8442c18c987b195364fd51f359cc8) version of `unbond_public` to make it easier to read.

## Tool used

Manual Review

## Recommendation
Add a a conditional to check `r7` ( this is set to true if caller == stakers withdrawal address). If this is set and a smaller amount of bond reduction is supplied than the total (as the function argument `microcredits` then consider this a delegator withdrawal and handle the arithmetic correctly. This will allow validators to still force unbond and will prevent delegators from accidentally unbonding everything without meaning to.



## Discussion

**evanmarshall**

This is a real issue. I would categorize it as minor. Why it's minor is due to the factor that a single user (with both a validator and a delegator must set it up this way) and this single user has the ability to migrate to use separate withdrawal addresses.

# Issue M-3: `bond_validator` and `bond_public` do not allow bonding even if unbonding window has passed or there are no credits unbonding 

Source: https://github.com/sherlock-audit/2024-05-aleo-judging/issues/36 

## Found by 
dtheo
## Summary

`bond_validator` and `bond_public` do not allow bonding if there currently exists an entry in the global `unbonding[]` map for the validator. This will fail even if the unbonding window is over, incorrectly preventing valid bonding in these cases.

## Vulnerability Detail
`bond_validator` and `unbond_validator` do not allow bonding if there currently exists an entry in the global `unbonding[]` map for the validator. This will fail even if the unbonding window is over, incorrectly preventing valid bonding in these cases. The only way to re-allow bonding is to call `claim_unbond_public` on the validator. It is also possible for delegators and validators to have a 0 balance for their unbonding credits (see issue #3). This means that invalid unbonding logic can prevent valid bonding flows in multiple ways when this `unbonding[]` check method is used.

## Impact

Delegators and validators attempting to make valid bonds will be unable to

## Code Snippet

[snarkVM/synthesizer/program/src/resources/credits.aleo#L391-L394](https://github.com/sherlock-audit/2024-05-aleo/blob/main/snarkVM/synthesizer/program/src/resources/credits.aleo#L391-L394)

[snarkVM/synthesizer/program/src/resources/credits.aleo#L265-L268](https://github.com/sherlock-audit/2024-05-aleo/blob/main/snarkVM/synthesizer/program/src/resources/credits.aleo#L265-L268)

## Tool used

Manual Review

## Recommendation

Change these checks to check if the unbonding window has passed, not just check if an unbonding entry exists as it may not yet be claimed.



## Discussion

**evanmarshall**

Duplicate of: https://github.com/sherlock-audit/2024-05-aleo-judging/issues/23

