# token-compliance-engine
A deterministic compliance analysis engine for tokenized assets. The engine reports compliance-relevant on-chain findings for regulated institutions without making legal conclusions.

## Phase 1 scope
- Ethereum mainnet only (ERC-20 contracts)
- Deterministic, verifiable on-chain checks (no mock data)
- Findings marked UNKNOWN when evidence is unavailable

## Implemented checks
- Upgradeability / proxy detection (EIP-1967)
- Ownership & admin control (Ownable `owner()`)
- Supply mechanics (total supply, cap when available)
- Transfer restrictions (pause state when available)

## Quick start
Set an Ethereum mainnet RPC endpoint:
```bash
export ETH_RPC_URL="https://mainnet.example"
```

Run the CLI:
```bash
python cli.py 0x0000000000000000000000000000000000000000
```

Output formats:
```bash
python cli.py 0x0000000000000000000000000000000000000000 --format json
```

## ABI-aware analysis (optional)
Provide an explicit ABI to enable token-specific checks (no ABI auto-fetching):
```bash
python cli.py 0x0000000000000000000000000000000000000000 --abi-file path/to/abi.json
```

Use a bundled ABI:
```bash
python cli.py 0x0000000000000000000000000000000000000000 --abi-name usdc
```

When ABI coverage is incomplete or a method cannot be verified, findings are marked UNKNOWN or FAILED.
Reports label each finding as `ONCHAIN` or `ABI` to distinguish the evidence source.

## Jurisdiction analysis (optional)
Jurisdiction modules map existing findings to regulatory expectations without making legal claims.
Each assessment is evidence-first and conservative: missing evidence yields INSUFFICIENT_EVIDENCE.

Example:
```bash
python cli.py 0x0000000000000000000000000000000000000000 --jurisdiction us-sec --jurisdiction eu-mica
```
Supported codes: `us-sec`, `eu-mica`.

## Example Analysis Output (USDC)
Below is a real deterministic compliance analysis generated against Ethereum mainnet
using ABI-aware inspection and jurisdictional evaluation (US SEC + EU MiCA). The output
is intended to be auditable and reproducible from the same inputs.

Command executed:
```bash
export ETH_RPC_URL="https://eth-mainnet.g.alchemy.com/v2/..."
python src/main.py \
  0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 \
  --abi-name usdc \
  --jurisdiction us-sec \
  --jurisdiction eu-mica
```

```bash

Token:
  Address: 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48
  Chain ID: 1
  Name: USD Coin
  Symbol: USDC
  Decimals: 6
  Total Supply: 51966229907068857
  Standard: ERC-20

Findings:
  - [ABI] [VERIFIED] [LOW] Pause state observed (ABI) (abi.transfer.pause_state)
    paused() returned a value via ABI.
    Evidence: method=paused(), raw_return=0x0000000000000000000000000000000000000000000000000000000000000000, interpretation=paused False
  - [ONCHAIN] [VERIFIED] [LOW] No EIP-1967 proxy detected (onchain.proxy.not_detected)
    EIP-1967 storage slots are empty. Upgradeability is not indicated.
    Evidence: method=eth_getStorageAt (EIP-1967 slots), raw_implementation=0x0000000000000000000000000000000000000000000000000000000000000000, raw_admin=0x0000000000000000000000000000000000000000000000000000000000000000, raw_beacon=0x0000000000000000000000000000000000000000000000000000000000000000, interpretation=all EIP-1967 slots are zero
  - [ABI] [VERIFIED] [MEDIUM] Admin controlled by EOA (ABI) (abi.ownership.eoa)
    ABI owner method returned an EOA address.
    Evidence: method=owner(), raw_return=0x000000000000000000000000fcb19e6a322b27c06842a71e8c725399f049ae3a, interpretation=owner has no bytecode (EOA)
  - [ABI] [UNKNOWN] [MEDIUM] Burn authority unknown (ABI) (abi.supply.burn_unknown)
    Burn methods not available in ABI.
    Evidence: method=burn/burnFrom, raw_return=none, interpretation=burn methods not present in ABI
  - [ABI] [VERIFIED] [MEDIUM] Mint authority mechanism detected (ABI) (abi.supply.mint_authority)
    Minter administration methods returned values via ABI.
    Evidence: method=isMinter(0x0000000000000000000000000000000000000000), raw_return=0x0000000000000000000000000000000000000000000000000000000000000000, interpretation=probe 0x0000000000000000000000000000000000000000 returned False, method=masterMinter(), raw_return=0x000000000000000000000000e982615d461dd5cd06575bbea87624fda4e3de17, interpretation=masterMinter returned address
  - [ABI] [VERIFIED] [MEDIUM] Blacklist capability detected (ABI) (abi.transfer.blacklist)
    isBlacklisted(address) returned a value via ABI.
    Evidence: method=isBlacklisted(0x0000000000000000000000000000000000000000), raw_return=0x0000000000000000000000000000000000000000000000000000000000000000, interpretation=probe 0x0000000000000000000000000000000000000000 returned False
  - [ABI] [UNKNOWN] [MEDIUM] Whitelist capability unknown (ABI) (abi.transfer.whitelist_unknown)
    isWhitelisted(address) not available in ABI.
    Evidence: method=isWhitelisted(0x0000000000000000000000000000000000000000), raw_return=none, interpretation=isWhitelisted not present in ABI, error=method not present in ABI
  - [ONCHAIN] [UNKNOWN] [MEDIUM] Supply cap status unknown (onchain.supply.cap_unknown)
    cap() was not detected; capped vs uncapped is UNKNOWN.
    Evidence: method=cap() selector, raw_return=none, interpretation=cap() not detected or failed to decode
  - [ONCHAIN] [UNKNOWN] [MEDIUM] Total supply unknown (onchain.supply.unknown)
    totalSupply() did not return a value. Supply is UNKNOWN.
    Evidence: method=totalSupply() selector, raw_return=none, interpretation=totalSupply() not detected or failed to decode

Jurisdictions:
  - United States - SEC: RISK (One or more SEC-aligned dimensions indicate elevated risk.)
    - Issuer/Admin Control: RISK
      Evidence: abi.ownership.eoa
      Rationale: SEC scrutiny increases with centralized admin authority.
      Explanation: Admin control is exercised by an externally owned account.
    - Transfer Controls: RISK
      Evidence: abi.transfer.pause_state, abi.transfer.blacklist
      Rationale: SEC expectations require disclosure of issuer intervention in transfers.
      Explanation: Transfer restriction capability detected.
    - Supply Governance: RISK
      Evidence: abi.supply.mint_authority
      Rationale: SEC guidance emphasizes disclosure of mint/burn authority and supply integrity.
      Explanation: Supply control mechanisms or inconsistencies detected.
    - Upgradeability: PASS
      Evidence: onchain.proxy.not_detected
      Rationale: Reduced upgrade authority lowers issuer control concerns.
      Explanation: No EIP-1967 upgradeability detected.
  - European Union - MiCA: RISK (One or more MiCA-aligned dimensions indicate elevated risk.)
    - Issuer/Admin Control: RISK
      Evidence: abi.ownership.eoa
      Rationale: MiCA expects governance transparency for issuer-controlled tokens.
      Explanation: Admin control is exercised by an externally owned account.
    - Transfer Controls: RISK
      Evidence: abi.transfer.pause_state, abi.transfer.blacklist
      Rationale: MiCA considers issuer intervention in transfers a governance risk.
      Explanation: Transfer restriction capability detected.
    - Supply Governance: RISK
      Evidence: abi.supply.mint_authority
      Rationale: MiCA requires disclosure of mint/burn controls and accurate supply data.
      Explanation: Supply control mechanisms or inconsistencies detected.
    - Upgradeability: PASS
      Evidence: onchain.proxy.not_detected
      Rationale: MiCA governance expectations favor immutable logic when disclosed.
      Explanation: No EIP-1967 upgradeability detected.

Notes:
  - Deterministic on-chain checks executed.
  - Findings marked UNKNOWN require additional ABI or off-chain evidence.
  - ABI-aware inspection enabled: usdc.

```
