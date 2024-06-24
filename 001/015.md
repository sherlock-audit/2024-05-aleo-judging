Shaggy Marmalade Cuckoo

High

# Using `self.signer` to verify identity may be used by attackers for "identity fraud" or "permission bypass"

## Summary
Using `self.signer` to verify identity may be used by attackers for "identity fraud" or "permission bypass"
## Vulnerability Detail
Quoting the definition of `self.signer` in the documentation:
>self.signer
>The self.signer command returns the user address that originated the transition. This can be useful for managing access control to a program. In the above example, the transfer_public_to_private function decrements the balance of the sender publicly using self.signer.


Using `self.signer` for permission verification is a very dangerous thing.
Taking `transfer_public_as_signer` as an example, the attacker can deploy the `Attack.aleo` contract and point one of its methods to `credits.aleo::transfer_public_as_signer`. At this time, because `credits.aleo::transfer_public_as_signer` uses `self.signer` as the authentication method. If the victim calls the method in `Attack.aleo`, the funds in his account will be transferred by the attacker.
```aleo
function transfer_public_as_signer:
    // Input the receiver.
    input r0 as address.public;
    // Input the amount.
    input r1 as u64.public;
    // Transfer the credits publicly.
@>    async transfer_public_as_signer self.signer r0 r1 into r2;
    // Output the finalize future.
    output r2 as credits.aleo/transfer_public_as_signer.future;
```


## Impact
Using unsafe `self.signer` may be used by attackers for "identity fraud" or "permission bypass"
## Code Snippet
https://github.com/sherlock-audit/2024-05-aleo/blob/55b2e4a02f27602a54c11f964f6f610fee6f4ab8/snarkVM/synthesizer/program/src/resources/credits.aleo#L159-L182
https://github.com/sherlock-audit/2024-05-aleo/blob/55b2e4a02f27602a54c11f964f6f610fee6f4ab8/snarkVM/synthesizer/program/src/resources/credits.aleo#L798-L806
https://github.com/sherlock-audit/2024-05-aleo/blob/55b2e4a02f27602a54c11f964f6f610fee6f4ab8/snarkVM/synthesizer/program/src/resources/credits.aleo#L1005-L1021
## Tool used

Manual Review

## Recommendation

`self.caller` better reflects the current caller and is a safer choice. It is recommended to use `self.caller` instead of `self.signer` for authentication. `self.signer` is only used when you really need to know the account that originally initiated the transaction, and it is not recommended as a method of authentication.
For example:
```diff
function transfer_public_as_signer:
    // Input the receiver.
    input r0 as address.public;
    // Input the amount.
    input r1 as u64.public;
    // Transfer the credits publicly.
-    async transfer_public_as_signer self.signer r0 r1 into r2;
+    async transfer_public_as_signer self.caller r0 r1 into r2;
    // Output the finalize future.
    output r2 as credits.aleo/transfer_public_as_signer.future;
```
