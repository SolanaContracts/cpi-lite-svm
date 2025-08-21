# cpi-lite-svm

<img width="1137" height="377" alt="Screenshot 2025-08-21 at 9 46 59 PM" src="https://github.com/user-attachments/assets/7db4fef3-bae4-4714-80b3-3540ecd1b3b9" />

A simple example showing the exact flow from your diagram: **Wallet → Middle Contract → Double Contract** using Cross-Program Invocation (CPI) without Anchor framework.

## What does this project do?

Following your diagram exactly:

1. **Wallet** sends a transaction to **Middle Contract**
2. **Middle Contract** receives the request and calls **Double Contract** (System Program)
3. **Double Contract** creates a new data account with 1 SOL
4. The **data account (2)** gets created successfully

This is **Cross-Program Invocation (CPI)** - where the Middle Contract calls the Double Contract on behalf of the Wallet.

## Folder Renaming Guide

Based on your diagram, here's what you should rename:

**Current Structure:**
```
├── contract/           # This is your "Middle Contract"
├── client/            # This is your "Wallet" 
└── basic-client/      # Rename this to "direct-wallet"
```

**Rename to match your diagram:**
```bash
# Rename the folders to match your image
mv contract/ middle-contract/
mv client/ wallet/  
mv basic-client/ direct-wallet/
```

**New Structure:**
```
├── middle-contract/    # The Middle Contract from your diagram
├── wallet/            # The Wallet from your diagram  
└── direct-wallet/     # Shows direct wallet→system program (no middle contract)
```

## Code Explanation Using Your Diagram Names

### 1. Middle Contract (`middle-contract/src/lib.rs`)

This is the **Middle Contract** from your diagram. Here's what it does:

```rust
pub fn process_instruction(
    program_id: &Pubkey,        // Middle Contract's ID
    accounts: &[AccountInfo],   // Accounts from Wallet
    _instruction_data: &[u8],   // Data from Wallet (empty)
) -> ProgramResult {
```

**Step 1: Receive accounts from Wallet**
```rust
let payer_account = next_account_info(iter)?;     // Wallet account
let pda_account = next_account_info(iter)?;       // Future data account (2)
let system_program = next_account_info(iter)?;    // Double Contract (System Program)
```

**Step 2: Create PDA for data account (2)**
```rust
let (pda, bump) = Pubkey::find_program_address(
    &[b"client1", payer_pubkey.as_ref()],  // Seeds for data account (2)
    &program_id,                           // Middle Contract controls it
);
```

**What happens here:**
- Middle Contract creates a special address (PDA) for **data account (2)**
- This PDA can only be controlled by Middle Contract
- The address is predictable using: "client1" + Wallet address + Middle Contract ID

**Step 3: Middle Contract calls Double Contract**
```rust
let ix = create_account(
    &payer_account.key,    // Wallet pays
    &pda,                  // Address for data account (2)
    1000000000,            // Put 1 SOL in data account (2)
    4,                     // 4 bytes of storage
    &SYSTEM_PROGRAM_ID,    // Double Contract (System Program) owns it
);
```

**Step 4: Execute the call to Double Contract (CPI)**
```rust
let signer_seeds = &[b"client1", payer_pubkey.as_ref(), &[bump]];
invoke_signed(&ix, accounts, &[signer_seeds])?;
```

**This is the CPI!** Middle Contract is calling Double Contract (System Program) to create **data account (2)**.

### 2. Wallet (`wallet/index.test.ts`)

This represents the **Wallet** from your diagram:

```typescript
// Wallet sets up the blockchain
liveSvm = new LiteSVM();
programId = PublicKey.unique();  // Middle Contract ID
payer = Keypair.generate();      // Wallet keypair

// Wallet deploys Middle Contract
liveSvm.addProgramFromFile(programId, "./contract.so");

// Wallet gets some SOL
liveSvm.airdrop(payer.publicKey, BigInt(100000000000));

// Wallet calculates where data account (2) will be created
[pda, bump] = PublicKey.findProgramAddressSync(
  [Buffer.from("client1"), payer.publicKey.toBuffer()], 
  programId  // Middle Contract ID
);
```

**Wallet creates transaction to Middle Contract:**
```typescript
let ix = new TransactionInstruction({
  keys: [
    { pubkey: payer.publicKey, isSigner: true, isWritable: true },    // Wallet
    { pubkey: pda, isSigner: false, isWritable: true },              // data account (2)
    { pubkey: SystemProgram.programId, isSigner: false, isWritable: false } // Double Contract
  ],
  programId,           // Middle Contract ID
  data: Buffer.from("") // No data needed
});
```

**Wallet sends transaction:**
```typescript
const tx = new Transaction().add(ix);
liveSvm.sendTransaction(tx);

// Wallet checks if data account (2) was created with 1 SOL
const balance = liveSvm.getBalance(pda);
expect(Number(balance)).toBe(1000000000);
```

### 3. Direct Wallet (`direct-wallet/index.ts`)

This shows what happens when **Wallet** talks directly to **Double Contract** (skipping Middle Contract):

```typescript
// Direct Wallet → Double Contract (no Middle Contract)
const instruction = SystemProgram.createAccount({
    fromPubkey: kp.publicKey,           // Wallet pays
    newAccountPubkey: dataAccount.publicKey, // Regular account (not PDA)
    lamports: 1000_000_000,             // 1 SOL
    space: 8,                           // Storage space
    programId: SystemProgram.programId   // Double Contract owns it
});

await conn.sendTransaction(tx, [kp, dataAccount]);
```

## The Flow From Your Diagram

1. **Wallet** creates a transaction
2. **Wallet** sends transaction to **Middle Contract** 
3. **Middle Contract** receives the transaction
4. **Middle Contract** calls **Double Contract** (System Program) via CPI
5. **Double Contract** creates **data account (2)** with 1 SOL
6. Success! **data account (2)** now exists

## Key Difference: PDA vs Regular Account

**With Middle Contract (your main project):**
- Creates **data account (2)** at a PDA address
- Only **Middle Contract** can control this account
- Address is predictable and unique

**Direct Wallet (simple version):**
- Creates regular account
- **Wallet** controls the account directly
- Uses random keypair for address

## After Renaming Your Folders

```
├── middle-contract/        # Your Middle Contract (Rust program)
│   ├── src/lib.rs         # Main contract logic
│   └── Cargo.toml
├── wallet/                # Your Wallet (test client)
│   ├── index.test.ts      # Wallet test
│   ├── contract.so        # Compiled Middle Contract
│   └── package.json
└── direct-wallet/         # Direct Wallet example
    ├── index.ts           # Direct System Program call
    └── package.json
```

## How to Run (After Renaming)

1. **Build Middle Contract:**
   ```bash
   cd middle-contract
   cargo build-bpf
   cp target/deploy/empty-contract.so ../wallet/contract.so
   ```

2. **Test Wallet → Middle Contract → Double Contract:**
   ```bash
   cd wallet
   bun install
   bun test
   ```

3. **Try Direct Wallet → Double Contract:**
   ```bash
   cd direct-wallet
   bun install
   bun run index.ts
   ```
