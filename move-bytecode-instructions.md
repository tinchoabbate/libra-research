# Move bytecode instructions

A table with all Move bytecode instructions as shown in ["Move: a language with programmable resources"](https://developers.libra.org/docs/assets/papers/libra-move-a-language-with-programmable-resources.pdf).

|Instruction |Description|May fail at runtime|
|------------|-----------|-------------------|
|`MoveLoc<x>`|Push value stored in x on to the stack. `x` is now unavailable|No|
|`CopyLoc<x>`|Push value stored in `x` on to the stack|No|
|`StoreLoc<x>`|Pop the stack and store the result in `x`. `x` is now available|No|
|`BorrowLoc<x>`|Create a reference to the value stored in `x` and push it on the stack|No|
|`ReadRef`|Pop `r` and push `*r` on the stack|No|
|`WriteRef`|Pop two values `v` and `r`, perform the write `*r = v`|No|
|`ReleaseRef`|Pop `r` and decrement the appropriate refcount if `r` is a global reference|No|
|`FreezeRef`|Pop mutable reference `r`, push immutable reference to the same value|No|
|`Call<p>`|Pop arguments `r*`, load into `p`'s formals `x*`, transfer control to `p`|No|
|`Return`|Return control to the previous frame in the call stack|No|
|`Pack<n>`|Pop arguments `v*`, create struct of type `n` with `f_i: v_i`, push it on the stack|No|
|`Unpack<n>`|Pop struct of type `n` from the stack and push its fields values `v*` on the stack|No|
|`BorrowField<f>`|Pop reference to a struct and push a reference field `f` of the struct|No|
|`MoveToSender<n>`|Pop resource of type `n` and publish it under the sender's address|Yes|
|`MoveFrom<n>`|Pop address `a`, remove resource of type `n` from `a`, push it|Yes|
|`BorrowGlobal<n>`|Pop address `a`, push a reference to the resource of type `n` under `a`|Yes|
|`Exists<n>`|Pop address `a`, push bool encoding "a resource of type `n` exists under `a`"|No|
|`GetGasRemaining`|Push u64 representing remaining gas unit budget|No|
|`GetTxnSequenceNumber`|Push u64 encoding the transaction's sequence number|No|
|`GetTxnPublicKey`|Push byte array encoding the transaction sender's public key|No|
|`GetTxnSenderAddress`|Push address encoding the sender of the transaction|No|
|`GetTxnMaxGasUnits`|Push u64 representing the initial gas unit budget|No|
|`GetTxnGasUnitPrice`|Push u64 representing the Libra coin per gas unit price|No|
|`PopUnrestricted`|Pop a non-resource value|No|
|`RejectUnless`|Pop bool `b` and u64 `u`, fail with error code `u` if `b` is `false`|Yes|
|`CreateAccount`|Pop address `a`, create a `LibraAccount.T` under `a`|Yes|
|`LoadTrue`|Push `true` on the stack|No|
|`LoadFalse`|Push `false` on the stack|No|
|`LoadU64<u>`|Push the u64 `u` on the stack|No|
|`LoadAddress<a>`|Push the address `a` on the stack|No|
|`LoadBytes<b[]>`|Push the byte array `b[]` on the stack|No|
|`Not`|Pop boolean `b` and push `¬b`|No|
|`Add`|Pop two u64's `u1` and `u2` and push `u1 + u2`. Fail on overflow.|Yes|
|`Sub`|Pop two u64's `u1` and `u2` and push `u1 - u2`. Fail on underflow.|Yes|
|`Mul`|Pop two u64's `u1` and `u2` and push `u1 x u2`. Fail on overflow.|Yes|
|`Div`|Pop two u64's `u1` and `u2` and push `u1 / u2`. Fail if `u2` is zero.|Yes|
|`Mod`|Pop two u64's `u1` and `u2` and push `u1 mod u2`. Fail if `u2` is zero.|Yes|
|`BitOr`|Pop two u64's `u1` and `u2` and push `u1 | u2`|No|
|`BitAnd`|Pop two u64's `u1` and `u2` and push `u1 & u2`|No|
|`Xor`|Pop two u64's `u1` and `u2` and push `u1 ⊕ u2`|No|
|`Lt`|Pop two u64's `u1` and `u2` and push `u1 < u2`|No|
|`Gt`|Pop two u64's `u1` and `u2` and push `u1 > u2`|No|
|`Le`|Pop two u64's `u1` and `u2` and push `u1 <= u2`|No|
|`Ge`|Pop two u64's `u1` and `u2` and push `u1 >= u2`|No|
|`Or`|Pop two booleans `b1` and `b2` and push `b1 ∨ b2`|No|
|`And`|Pop two booleans `b1` and `b2` and push `b1 ∧ b2`|No|
|`Eq`|Pop two values `r1` and `r2` and push `r1 = r2`|No|
|`Neq`|Pop two values `r1` and `r2` and push `r1 ≠ r2`|No|
|`Branch<l>`|Jump to instruction `l` in the current procedure|No|
|`BranchIfTrue<l>`|Pop boolean, jump to instruction index `l` in the current procedure if `true`|No|
|`BranchIfFalse<l>`|Pop boolean, jump to instruction index `l` in the current procedure if `false`|No|