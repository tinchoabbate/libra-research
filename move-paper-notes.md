# Move paper notes

Link to paper: https://developers.libra.org/docs/assets/papers/libra-move-a-language-with-programmable-resources.pdf

> Move is a programming language for implementing custom transaction logic and smart contracts in the Libra protocol

## Intro
- The role of a blockchain programming language is to decide how transitions and state are represented.

## Move design goals

### First-class assets (resources)

- The key feature of Move is the ability to define custom _resource types_. A resource can never be copied or discarded, only moved between program storage locations.
- Safety guarantees enforced statically by Move's type system
- Libra coin is implemented as an ordinary Move resource
- Resource is declared inside a _module_.
- Module declares one or many resource types and procedures that state how its resources are created, destroyed, updated.
- A module can invoke other modules' procedures and use their types.

### Flexibility
- Thanks to _transaction scripts_ and Move modules
- A tx script is a single procedure with arbitrary Move code => customizable transactions
- A tx script can invoke procedures of modules published
- Tx scripts are atomic transactions
- A module is a long-lived piece of code published in the global state

### Safety
- Move _must_ reject programs that do not satisfy key properties: resource safety, type safety and memory safety.
- Executable format of Move is a typed bytecode (higher than assembly, lower than source language).
- Bytecode is checked for resource, type and memory safety by a "bytecode verifier" and then executed by an interpreter

### Verifiability
Approaches to make Move more verification-friendly:

1. No dynamic dispatch => target of each call is statically determined. Easier for verification tools.
2. Limited mutability: every mutation to a Move value occurs through a reference (temporary value created a destroyed in the same tx). Bytecode verifier ensures at most one reference for a value exists at any point in time.
3. Modularity: modules enforce data abstraction and encapsulate critical operations on resources.

## Overview

- Each read of a Move variable `x` must specify whether:
  - A) The usage moves `x`'s value with `move(...)`, rendering `x` unavailable
  - or B) copies the value with `copy(...)`, leaving `x` available for continued use
- Unrestricted types (such as `u64` and `address`) can be handled either with `move` or `copy`.
- Resources can _only_ be `move`d.
- Failing to `move` a resource when you must should trigger a bytecode verification error. For instance, you must make a deposit after a withdrawal of Libra coins.

- Each account can contain 0 or more modules and 1 or more resources
- An account can contain at most 1 resource of a type and at most 1 module with a given name
- An account can hold multiple instances of a given resource types by wrapping: `resource TwoCoins {c1: 0x0.Currency.Coin, c2: 0x0.Currency.Coin }`
- The address of the declaring module is part of a type => `0x0.Currency.Coin` and `0x1.Currency.Coin` are different types

### The `Currency` module

See all modules in Move's standard library [here](https://github.com/libra/libra/tree/master/language/stdlib/modules).

#### Declaring the module and a resource

The following module is named `Currency` and declares a resource type called `Coin`:

~~~
module Currency {
  resource Coin { value: u64 }
  // ...
}
~~~

- `Coin` is a struct type with a single field of type `u64`
- The structure of `Coin` is opaque outside the module. This means:
  - Othe modules / tx scripts can only write or reference the `value` field via the module's public procedures
  - Only the module's procedures can create or destroy values of type `Coin`
- Outside of the API exposed by this module, `move` is the only operation another module can perform on a `Coin`

#### Deposit

~~~
public deposit(payee: address, to_deposit: Coin) {
  // Destroy the input Coin and record its value
  // Unpack<T> takes a resource, destroys it, and returns the values bound to the fields of the resource
  // Unpack<T> only works on resources declared in the current module
  let to_deposit_value: u64 = Unpack<Coin>(move(to_deposit));

  // Acquire a reference to the unique Coin resource in `payee` account
  // BorrowGlobal<T> returns a reference to the unique instance of T published in the passed address
  // `coin_ref` is a mutable reference to a Coin resource
  let coin_ref: &mut Coin = BorrowGlobal<Coin>(move(payee));

  // Increment the value of Coin in `payee` by the value of the Coin passed as argument
  // first obtaining a reference to the Coin's value field
  let coin_value_ref: &mut u64 = &mut move(coin_ref).value;
  let coin_value: u64 = *move(coin_value_ref);
  *move(coin_value_ref) = move(coin_value) + move(to_deposit_value);
}
~~~

#### Withdraw

~~~
public withdraw_from_sender(amount: u64): Coin {
  // Get a reference to the unique resource of type Coin published in sender's account
  let transaction_sender_address: address = GetTxnSenderAddress();
  let coin_ref: &mut Coin = BorrowGlobal<Coin>(move(transaction_sender_address));

  // Decrease the value of the referenced Coin by `amount`,
  // ensuring sender cannot withdraw more than what they have
  let coin_value_ref: &mut u64 = &mut move(coin_ref).value;
  let coin_value: u64 = *move(coin_value_ref);
  RejectUnless(copy(coin_value) >= copy(amount)); // Reverts entire tx upon failure
  *move(coin_value_ref) = move(coin_value) - copy(amount);
  
  // Create and return a new resource of type Coin with value `amount`
  let new_coin: Coin = Pack<Coin>(move(amount));
  // Resource is moved to the caller => caller now owns this Coin resource
  return move(new_coin);
}
~~~

## The Move Language in detail

### Global state
- The global state is organized as a partial map from addresses to accounts
- Accounts contain both resource data values and module code values
- Different resources in an account must have distinct identifiers
- Different modules in an account must have distinct names

### Modules
- A module consists of:
  - Name
  - Struct declarations (including resources)
  - Procedure declarations
- Module's code can refer to a published module using a unique identifier (module's account address + module's name)
- Procedures of a module declare rules for creating / writing / destroying the types declared by the module

### Types
- Primitive types:
  - Boolean
  - Unsigned integers (64 bits)
  - Address (256 bits)
  - Fixed-size byte arrays

- Struct: is a user-defined type declared by a module. Declared as a resource with `resource`
- All primitives and non-resource structs are "unrestricted" types
- Variables can be "resource" or "unrestricted" type
- Resource variables:
  - Cannot be copied. Must be `move`d.
  - Cannot be reassigned. Otherwise is destroyed.
  - A reference cannot be de-referenced (since this would create a copy)
- Unrestricted variables:
  - Can be copied
  - Can be reassigned
  - Can be dereferenced
  - If it's a struct, may not contain a field with a resource type

### Values
- Move supports struct, primitive and reference values.
- A struct cannot contain a reference field
- Reference values are _transient_ => a reference must be created and destroyed during execution of a tx script
- References cannot reference other references. Only primitive and structs.
- How to obtain references:
  - `BorrowLoc` reference to a local variable
  - `BorrowField` reference to a struct's field (only on struct type declared in current module)
  - `BorrowGlobal` reference to a resource published under an account (only on struct type declared in current module)

### Procedures and transaction scripts

- Procedure signature: visibility + typed formal params + return types
- A procedure declaration contains:
  - Signature
  - Typed local variables
  - Array of bytecode instructions
- Procedure can be uniquely identified with module identifier + signature
  - All procedure calls in Move are statically determined
- Procedure visibility can be `public` or `internal`
  - Public: invoked by any module or tx script
  - Internal: only invoked within module

- A module can only depend on modules that were published earlier in the linear tx history

- A transaction script is a procedure declaration with no associated module

## Bytecode interpreter

- Stack-based interpreter for Move bytecode
- Interpreter supports procedure calls:

  1. Caller pushes arguments to a procedure onto the stack
  2. Caller invokes `Call` instruction
  3. A new call stack frame is created for the callee, where pushed values are loaded into the callee's local variables
  4. Bytecode interpreter begins executing the instructions

- Execution is _metered_ (similar to EVM)
- Each instruction costs _gas_
- Interpreter tracks gas units remaining and halts if reaches zero

### Instructions
6 broad types of instructions:

- Copying and moving data from local vars to stack (e.g. `CopyLoc`, `MoveLoc`) and moving data from stack to local vars (e.g. `StoreLoc`)
- Operations on typed stack values: pushing constants, arithmetic / logic operations
- Module builtins, such as:
  - `Pack` and `Unpack` for creating and destroying module's declared types
  - `MoveToSender` and `MoveFrom` for publishing and unpublishing the module's type under an account
  - `BorrowField` for acquiring a reference to a field of one of the module's types
- Reference related instructions, such as:
  - `ReadRef` for reading references
  - `WriteRef` for writing references
  - `ReleaseRef` for destroying a reference
  - `FreezeRef` for turning a mutable into an immutable reference
- Control-flow operations (conditional branching, calling, returning)
- Blockchain-specific builtin operations: getting caller's address, creating a new account

See all instructions in [the table of Move bytecode instructions](./move-bytecode-instructions.md).

#### Native modules
- Some instructions (e.g. `sha3`) could be provided at bytecode-level. Instead, they're mplemented as modules in the "standard library".
- Standard library procedures are declared as `native`
- `native` procedures' bodies are provided by the Move VM

## Bytecode verifier
- Statically enforces safety properties for modueles and tx scripts
- All Move programs must go through the verifier
- Categories for checks:
  - Structural checks: bytecode is well-formed
  - Semantic checks on procedure bodies: incorrect arguments, dangling references, duplicating resources
  - Linking uses of struct and procedures: ilegally invoking internal procedures, using procedures that do not match declaration

### Control-flow graph
- Verifier constructs a control-flow graphg by decomposing the instructions into a collection of basic blocks.
- Each basic block ends with branch or return instruction

### Stack balance checking
- Ensures callee cannot access stack locations that belong to callers

### Type checking
- Ensures each instruction and procedure is invoked with arguments of appropriate types
- Types of local variables of a procedure are provided in bytecode
- Types of stack values are inferred

#### Kind checking
Additional checks in type checking:
- Resources cannot be duplicated
- Resources cannot be destroyed
- Resources must be used

### Reference checking
- All references must point to allocated storage (i.e. no dangling references)
- All references must have safe read / write access
- Reference checking scheme has "novel features" that will be a topic of a separate paper

### Linking with global state
- Linker checks that:
  - Used struct declarations match name and kind
  - Used procedure signatures match name + visibility + formal parameter types + return types

## Move next steps
- Implementing core Libra functionality: accounts, reserve management, addition / removal of validator nodes, handling of tx fees, cold wallets.
- New language features
  - Generics, collections and events
  - Mechanism for versioning and updating modules, tx scripts and published resources
- Improved DX
  - High-level source language
  - Improvements in IR
- Formal specs and verification
  - A logical spec language and automated formal verification tool
- Third-party Move modules
  - Creating a marketplace for high-assurance modules and providing effective tools for veryfing Move code
