# CLAUDE.md

## Important
Think carefully and only action the specific task I have given you with the most concise and elegant solution that changes as little code as possible.

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Solana arbitrage bot program that identifies and executes arbitrage opportunities between two swap programs. The project consists of:
- A Solana on-chain program (Rust) in `/program`
- A Next.js web UI in `/app`
- Test suite for the arbitrage program in `/tests`

## Build and Development Commands

### Solana Program (Rust)
```bash
# Build the Solana program
cd program && cargo build-sbf

# Deploy the program (requires configured Solana CLI)
solana program deploy target/deploy/arb_program.so

# Format Rust code
cargo fmt

# Check Rust code
cargo check
```

### Frontend Application
```bash
# Install dependencies
cd app && yarn install

# Run development server
yarn dev

# Build for production
yarn build

# Run linter
yarn lint
```

### Testing
```bash
# Run tests (from root directory)
yarn test

# Lint all TypeScript/JavaScript files
yarn lint
yarn lint:fix  # Auto-fix formatting issues
```

## Architecture

### Program Structure
The arbitrage program (`/program`) is a native Solana program (not using Anchor) with:
- **Entry point**: `program/src/lib.rs` - Defines the `TryArbitrage` instruction
- **Processor**: `program/src/processor.rs` - Core arbitrage logic that validates pools and executes trades
- **Arbitrage logic**: `program/src/arb.rs` - Algorithm for detecting arbitrage opportunities
- **Swap calculations**: `program/src/swap.rs` - Constant-product algorithm implementation
- **Partial state**: `program/src/partial_state.rs` - Optimized deserialization using bytemuck for compute efficiency

Key design decisions:
- Returns error when no arbitrage opportunity exists (leverages preflight checks to avoid transaction fees)
- Uses partial deserialization to optimize compute units
- Supports configurable concurrency (number of assets to evaluate) and temperature (aggressiveness threshold)

### Client Architecture
- **Frontend**: Next.js app with Solana wallet integration
- **Tests**: Use Address Lookup Tables to pack more accounts into transactions
- **Instruction format**: Accounts must be provided in specific order (see processor comments)

## Key Configuration Parameters

- **Concurrency**: Number of assets to evaluate combinations across (1-n)
- **Temperature**: Aggressiveness of arbitrage identification (0-99, higher = more aggressive)

## Account Ordering Requirements

Accounts must be provided to the program in this exact order:
1. Payer
2. Token Program
3. Swap #1 Liquidity Pool
4. Swap #2 Liquidity Pool
5. [Token Accounts for User] (count = concurrency)
6. [Token Accounts for Swap #1] (count = concurrency)
7. [Token Accounts for Swap #2] (count = concurrency)
8. [Mint Accounts] (count = concurrency)

## Testing Setup

Before running tests:
1. Deploy two swap programs (or use existing ones)
2. Update program IDs in `tests/main.test.ts`:
   - `SWAP_PROGRAM_1`
   - `SWAP_PROGRAM_2`
3. Configure test parameters:
   - `temperature`: Arbitrage aggressiveness
   - `concurrency`: Assets evaluated per instruction
   - `iterations`: Number of times to check all pairings

## Important Implementation Details

- Program uses CPI (Cross-Program Invocation) to execute swaps on external programs
- Leverages Solana's preflight simulation to only pay fees for profitable trades
- Uses Address Lookup Tables in tests to reduce transaction size
- Implements partial deserialization with bytemuck for compute optimization
- Pool addresses are validated using PDA derivation with seed "liquidity_pool"