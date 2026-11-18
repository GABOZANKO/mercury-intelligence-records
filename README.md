# Mercury Intelligence records

Mercury Intelligence is the allocation model of Mercury Alpha Reserve. Once a day it decides how the fund should be split across its layers (reserve, income, systematic alpha, gold, bitcoin, frontier) and which carry trades to hold. It publishes that decision on Solana with two hashes: the sha256 of its **inputs** (the market data it used) and of its **rationale** (why it decided what it did).

This repository holds the files behind those hashes, so anyone can check that what the model says it saw and reasoned matches what it committed to on chain, at the time it committed.

The model currently runs in **shadow mode on Solana devnet**: decisions are published and tracked as a paper portfolio, but move no funds.

## Layout

```
records/<YYYY-MM-DD>/
  inputs.json      market data the decision used (sha256 = on-chain inputs_hash)
  rationale.json   signals, rules applied and resulting weights (sha256 = on-chain rationale_hash)
  params.json      the decision exactly as published: targets and hashes
  run.json         the run result: decision account, transaction signature, status
```

One folder per UTC day with a published decision. A missing day means no decision was published that day.

`inputs.json` and `rationale.json` are canonical JSON: their exact bytes are what was hashed. Do not reformat them before hashing.

## On chain

| | |
|---|---|
| Cluster | devnet |
| Program | `rGBiv2WrKT3GULoZwkwEJqHT2xf1hFXkHToj6pY39DQ` |
| Model account | `9yy14qCoXwwsoU8tCi94zeswbzcirBp3YSMHEbDXpQdn` (`mercury-intel-v1`, version 1) |
| Model signer | `7ANQ1jFneEB7vZQVFf7uHZLgtoutd2emzDhUCSsqdR36` |

Each decision is stored in its own account (`AllocationDecisionState`, one per model per UTC day). Its address is in that day's `run.json` (`publish_result.decision`).

## Verify a day

1. **Hash the files.**

   ```bash
   cd records/2026-11-09
   sha256sum rationale.json inputs.json
   ```

2. **Open the decision account on Solana Explorer** (devnet) and check that it exists and is owned by the program above. Explorer shows the account's raw data but does not decode its fields, because the program does not publish its interface on chain.

3. **Read the two hashes from the account data.** In the account's raw bytes, `inputs_hash` is bytes 213–244 and `rationale_hash` is bytes 245–276 (32 bytes each, counting from 0). With the Solana CLI and Python:

   ```bash
   A=<decision account from run.json>
   solana account "$A" --url devnet --output json-compact > account.json
   python3 -c 'import base64,json; d=base64.b64decode(json.load(open("account.json"))["account"]["data"][0]); print("inputs_hash   ", d[213:245].hex()); print("rationale_hash", d[245:277].hex())'
   ```

4. **Compare.** The sha256 of `inputs.json` must equal `inputs_hash`, and the sha256 of `rationale.json` must equal `rationale_hash`. The transaction in `run.json` (`publish_result.signature`) shows on Explorer when the decision was published and that the model signer signed it.

## Worked example: 2026-11-09

The first decision of the model.

- Decision account: [`3dfQd5uHqDNwLLeup4hkeLqcZgfzoQNJ3hrZs6Ai5rzA`](https://explorer.solana.com/address/3dfQd5uHqDNwLLeup4hkeLqcZgfzoQNJ3hrZs6Ai5rzA?cluster=devnet)
- Transaction: [`5e3NF1J8…BxsHoX`](https://explorer.solana.com/tx/5e3NF1J8CLo9s1RFudDFZmBmJc9SeUMYZJrpq2nqF18j29s7UvBayKpN7L4fznrhKsi6EwfUMY5JrJNL1LBxsHoX?cluster=devnet)

| | sha256 of the file | read from the account |
|---|---|---|
| `inputs.json` | `3b0da44962babd23f0c6831b8578cf1ff039dc40b0334f7ef665b9e8cc4fd027` | bytes 213–244: same |
| `rationale.json` | `c868de4f119858cd76f5a3b517b3d8cf36b6a4cf675f5f67600b7cecc335df1d` | bytes 245–276: same |

The decision itself (from `params.json`, also in the account):

| Layer | Target |
|---|---|
| Reserve (idle USDC) | 20.00% |
| Income | 35.00% |
| Systematic alpha | 20.00%, of which SOL carry 18.34%, BTC 0, ETH 0 |
| Gold | 10.00% |
| Bitcoin | 10.00% |
| Frontier | 5.00% |

Data used up to 2026-11-09 00:00 UTC (`data_as_of`); published 2026-11-09 12:59 UTC (transaction time).

## What this is not

These are records of a model running in shadow mode on a test network. They are not investment advice, an offer, or a forecast.
