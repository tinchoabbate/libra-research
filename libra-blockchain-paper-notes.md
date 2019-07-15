# The Libra Blockchain paper notes

My personal notes for a quick and dirty summary on [The Libra Blockchain paper](https://developers.libra.org/docs/assets/papers/the-libra-blockchain.pdf). For more details on any section, refer to the corresponding section on the paper.

## Logical Data Model

### Versioned database

- All data is stored in a single versioned database.
- Version = `uint64` corresponding to the number of transactions the system executed
- At each version `i`: `(Ti,Oi,Si)` such that `Apply(Si-1, Ti)` --> `<Oi,Si>`. Means that executing transaction `Ti` against ledger state `Si-1` produces output `Oi` and a new ledger state `Si`
- Transactions are executed against latest version.
- Clients can query current and previous states.

### Ledger state

- Account-based data model to encode ledger state.
- State is a key-value store mapping [account address keys] --> [account values]
- Account address: 256 bits such that address = hash(public key). Private keys can be rotated without changing address.
- Account value: collection of Move resources (data) and modules (code)

#### Resource

- Every resource has a type declared by a module. A resource be contained by an account which may not have the declaring module.
- Resource type: nominal type consisting of `AddressOfModule.ModuleName.Type` of the declaring module. Ex.: `0x56.Currency.T` is the resource named `Currency.T` declared at `Currency` module, stored at address `0x56`.
- Each account can store at most one resource of a given type.
- A resource cannot be copied, only moved.
- A resource type can only be created or destroyed by the module that declares the type.
- Libra Coin is just another resource type in the Move language.

#### Module

- Contains Move bytecode. Declares resource types and procedures.
- Identified by the address of the account where the module is declared (e.g.` 0x56.Currency`).
- Uniquely named within an account.
- Modules are immutable (unless hard fork).

### Transaction

- Modifies ledger state after being committed by consensus.
- Consists of: transaction script (Move bytecode) and arguments to the script.
- Transaction output: execution status code, gas usage, and event list.

#### Events

- A tx generates events, but cannot read events => tx execution only depends on current state.
- Purpose of events is similar of logs and events in Ethereum, but mechanics in Libra are "quite different".

### Ledger history

- Sequence of committed and executed transactions (and the associated events emitted).
- In logical data model (theory), *no concept of "block of transactions"* => transactions occur in sequence without distinction by blocks.
- In practice, consensus protocol batches txs in blocks just as an optimization.

## Executing transactions

- It's the only way to change the blockchain state

### Execution requirements

- Genesis state must be agreed by all validators
- Genesis state must include core components of the blockchain defined as Move modules: logic of accounts, transaction validation, vaidator selection, Libra coin
- Initial state is empty. Genesis state is created from initial state using a special transaction (which defines specific modules and resources) and does not go through the normal tx process.
- Tx execution is hermetic and deterministic ==> only depends on TX's info and ledger's current state

#### Gas & Transaction fees

- Paid in Libra coins. Similar to Gas in Ethereum.
- Fee determined by: gas price and gas cost.
- Deduction of gas fees is coded in Move.
- VM disables metering of gas during execution of core components (to avoid circularity).

### Transaction structure

- Transaction is a signed message that contains:
  - Sender address: VM reads the sequence number, authentication key, and balance from the LibraAccount.T resource stored under this address
  - Sender public key: hash of this public key must match the authentication key stored under the sender’s LibraAccount.T resource
  - Program: a Move bytecode tx script to execute. Optionally: list of inputs to the script, list of Move bytecode modules to publish
  - Gas price: in Libra coins
  - Max gas amount
  - Sequence number: uint, must be equal to the sequence number from the sender’s LibraAccount.T resource. After this transaction executes, the sequence number is incremented by one. Prevents replay.
- It's said that a transaction "is an authenticated wrapperaround Move bytecode program"

### Steps to execute a transaction

- Execution is different from updating ledger's state ==> First execution, then consensus, then output written to ledger
- Six steps in the VM:

1) **Check signature**: tx's signature must match sender's public key and tx data. Does not require reading any data from sender's account.

2) **Run prologue** (gas metering disabled): it's a procedure of Move's `LibraAccount` module

- 2.1) Check `hash(sender's public key) == auth key` stored in sender's account
- 2.2) Check `gas price * max gas amount <= sender's account balance`
- 2.3) Check `tx's sequence number == sequence number in sender's account`

3) **Verify tx scripts and modules**: VM performs checks on tx script and modules using Move bytecode verifier (type-safety, reference-safety, resource-safety).

4) **Publish modules**: if any, published under the transaction's sender account. Fails if duplicate name.

5) **Run tx script**

6) **Run epilogue** (gas metering disabled): charge for gas used and increment sender's sequence number

- It's a procedure of Move's `LibraAccount` module
- Always run if execution advances beyond (2)

### The Move programming language

- Why move exists:
  - i) transaction scripts
  - ii) allow user-defined code and datatype (smart contracts) via modules
  - iii) support config and extensions of Libra protocol

- Key feature: ability to define custom "resource types", which are used to code programmable assets

#### Representations of Move programs

- Three representations: source code (not available yet), IR, bytecode
- Source and IR are compiled to bytecode
- Safety guarantees at the bytecode level: type-safety, reference-safety, resource-safety

#### Transaction scripts

- A script is an arbitrary Move bytecode program
- Script can invoke procedures of modules published. Example: a tx script that takes `recipient_address` and `amount` as arguments and invokes the available `LibraAccount.pay_from_sender(recipient_address, amount)` procedure is the same as an Ether transfer.

#### Modules

- A module is a code unit in the ledger state.
- A module declares (zero or many) struct types and procedures
- Procedures are like static methods of classes in OOP.
- Data fields cannot be accessed outside of its declaring module ==> No "public" fields
- Modules contain code. Resources contain data.
- A module is a recipe for creating resources. A module can create an arbitrary number of resources that can be published under different account addresses.
- For now, using Move only for system-defined modules, users cannot define their own modules.

##### Struct

- Contains primitive values (e.g. integer) or other structs
- Must be tagged as:
  - "resource" or
  - "unrestricted": can be copied or destructed. Cannot contain "resource" structs. Cannot be published under an account in state.

#### Move VM

- Implements a verifier and interpreter for Move bytecode
- Stack-based
- Executes transactions: first verifies, then runs the bytecode
- Supports types: booleans, uint64, address (32 bytes), fixed-size byte arrays, structs (including resources), and references
- Struct fields cannot be references => this prevents storage of references in the state
- No heap => local data is in stack and freed when procedure ends. Persistent data must be stored in state.

## Authenticated Data Structures and Storage

- Libra uses a single Merkle tree (Merkle Tree Accumulator) to provide an authenticated data structure for the entire ledger history ==> No idea of linked list of blocks.

### Accounts

- Logical level: collection of resources and modules stored under the account's address
- Physical level: ordered map of access paths (i.e. delimited strings) to byte array values
- An account is serialized as a list of access paths and values sorted by access path.
- The authenticator for an account is the hash of its serialized representation => Authenticator must be recomputed after any change to the account.
- Structures used to represent an account are optimized for small accounts with little use of storage.Libra acknowledges that storage growth associated with accounts can be a real problem in the future. They think of a rent-based model for storage.

## Consensus

- Libra uses a variant of the HotStuff consensus protocol called LibraBFT.
- LibraBFT assumes that a set of 3f + 1 votes is distributed among a set of validators that may be honest. LibraBFT remains safe, preventing attacks such as double spends and forks when at most f votes are controlled by honest validators.

### LibraBFT overview

- Validators receive and share transactions through a shared mempool protocol.
- LibraBFT proceeds in a sequence of rounds.
- In each round:
  - "Leader" and proposes a block of transactions to issue a certified sequence of blocks that contain the full previous transaction history
  - Validator receives the block and votes for certifying it (according to its rules)
  - To vote for the block, validator executes the set of transactions without external effect ==> Results in an "authenticator" for the database
  - Validator sends signed vote and authenticator to leader
  - Leader gathers votes to form a Quorum Certificate (QC) and broadcasts to all validators
- Block is considered committed with 3-chain confirmations (i.e. block K is committed after K+1 and K+2)
- Initially support 100 validators. In the future, 500-1000 validators.

### Validator management & epochs

- Libra protocol manages validators using a Move module.
- Each change to the set of validators defines an "epoch".
- Within an epoch, clients do not need to synchronize every QC. A client only needs to synchronize to the latest available QC in its current epoch.
- If a validator chooses to prune history as, it needs to retain at least enough data to provide proof of the validator set change to clients.
- The voting power must also remain honest for a period time after the epoch in order to allow clients to synchronize to the new configuration.
- A client that is offline for longer than this period needs to resynchronize using some external source of truth to acquire a checkpoint that they trust

## Ecosystem policies

### Libra Coin

- Reserve is managed by the Libra Association 
- There are authorized resellers who can transact large amounts of fiat and Libra in and out of the reserve.
- The Libra coin contract allows the Association to mint and burn coins according to demand.
- For new coins to be minted, there must be a commensurate fiat deposit in the reserve.
- The Libra coin contract is Move code, and can therefore be changed without modifying the underlying protocol of Libra. Additional functionality can be created, such as requiring multiple signatures to mint currency and creating limited-quantity keys to increase security.

### Validators

- Initially, the Libra Blockchain only grants votes to Founding Members.
- Aspire to make Libra fully permissionless
- Potential transition to PoS in the future. Transition requires
  - Ecosystem to be sufficiently large to prevent a single bad actor from causing disruption.
  - Existence of a competitive and reliable market for delegation for users that do not wish to become validators
  - Addressing technological and usability challenges in the staking of Libra coins.
