# MetaMask Money Account on Monad

A [HyperIndex](https://docs.envio.dev/docs/HyperIndex/overview) indexer for the vault behind [MetaMask's Money Account](https://support.metamask.io/trade/money-account/) on Monad. It follows every deposit, withdrawal and transfer of the vault's shares, so you get every wallet's balance, the day each wallet first deposited and new wallets per UTC day. Monad is on [HyperSync](https://docs.envio.dev/docs/HyperSync/overview), so the whole history (about 400K events since June) indexes in under a minute.

From the vault's first transfer on 2 Jun 2026 (Monad block 78,633,433) to Monad block 122,726,000 (11:16 UTC on 16 Nov 2026), **83.4K wallets have deposited into the Money Account**, and **about 800 new wallets make their first deposit every day**. From 2 to 15 Nov it was between 755 and 889 a day, 808 on average.

| | Count |
| --- | --- |
| Wallets that have deposited | 83,375 |
| Wallets holding vault shares | 69,374 |
| Wallets holding at least $1 | 38,870 |
| Deposits | 198,432 |
| Vault shares | 60,044,007 (about 61.5M mUSD) |
| mUSD the vault supplies to Aave v3 on Monad | 56.0M |
| Blocks covered | 78,633,433 to 122,726,000 |

New wallets in the last full week:

| Day (UTC) | New wallets |
| --- | ---: |
| 9 Nov | 768 |
| 10 Nov | 822 |
| 11 Nov | 755 |
| 12 Nov | 822 |
| 13 Nov | 881 |
| 14 Nov | 786 |
| 15 Nov | 788 |

The $1 line uses the share price on 16 Nov, 1.025068 mUSD per share, so 975,546 shares is $1. The Aave figure is `balanceOf` on aMonmUSD for the vault, read over RPC at the same block.

## Why it matters

The Money Account is a wallet feature, so most of its users never touch the vault directly. Every deposit still mints vault shares onchain, which means one `Transfer` event gives you the product's whole growth curve. A mint is a deposit, a burn is a withdrawal, and the first mint to a wallet marks a new user.

The same pattern works for any ERC-4626 style or Veda BoringVault vault. Index its share token and you get holders, balances, first deposit dates and daily signups without reading the vault's internals.

## What it indexes

| Contract | Chain | Address |
| --- | --- | --- |
| Money Account vault (Veda BoringVault, "mUSD Vault", vmUSD) | Monad (143) | [`0x1C8a336051D2024E318A229d01F9F6CF96efD316`](https://monadscan.com/address/0x1C8a336051D2024E318A229d01F9F6CF96efD316) |
| Accountant (share price) | Monad (143) | [`0x98A45D90E81849a5743241d3ff765F9Fd788206a`](https://monadscan.com/address/0x98A45D90E81849a5743241d3ff765F9Fd788206a) |
| mUSD | Monad (143) | [`0xacA92E438df0B2401fF60dA7E4337B687a2435DA`](https://monadscan.com/address/0xacA92E438df0B2401fF60dA7E4337B687a2435DA) |
| Aave v3 aMonmUSD | Monad (143) | [`0xDBb2e1b6BC94328Df2662A1796AB6A23fCde3199`](https://monadscan.com/address/0xDBb2e1b6BC94328Df2662A1796AB6A23fCde3199) |

The indexer only listens to the vault's `Transfer` event. The other three addresses are for reading the share price and the Aave position. The vault, accountant and mUSD addresses are MetaMask's own `moneyAccountVaultConfig` in the [MetaMask extension repo](https://github.com/MetaMask/metamask-extension/blob/main/test/e2e/feature-flags/feature-flag-registry.ts). Monad lists the same vault as `MoneyAccountVault` in its [protocols registry](https://github.com/monad-crypto/protocols/blob/main/mainnet/metamask.jsonc). The aToken comes from the [Aave address book](https://github.com/bgd-labs/aave-address-book/blob/main/src/AaveV3Monad.sol). Steakhouse Financial curates the vault and Veda built it ([Steakhouse](https://kitchen.steakhouse.financial/p/the-metamask-money-account), [MetaMask](https://metamask.com/news/introducing-metamask-money-account)).

The handler is [`src/handlers/MoneyAccountVault.ts`](src/handlers/MoneyAccountVault.ts). Entities in [`schema.graphql`](schema.graphql):

- `Totals` keeps wallets that have ever deposited, current holders, share supply, deposits and withdrawals
- `Holder` stores one row per wallet with its share balance, first deposit day and block, and deposit count
- `DailyStat` counts new wallets, deposits, shares minted and shares burned per UTC day

Shares have 6 decimals.

## Run it

You need Node 22+, pnpm and Docker, plus a free Envio API token from [envio.dev/app/api-tokens](https://envio.dev/app/api-tokens).

```bash
cp .env.example .env   # then paste your token into ENVIO_API_TOKEN
pnpm install
pnpm codegen
pnpm dev
```

There's no unit test suite yet. `pnpm full-run` is the closest thing, and it's covered under checking the numbers below.

GraphQL is at http://localhost:8080 (local admin secret `testing`).

```graphql
{
  Totals { wallets holders shareSupply deposits withdrawals lastBlock }
  DailyStat(order_by: { id: desc }, limit: 14) { id newWallets deposits }
  Holder(where: { balance: { _gte: "975546" } }, limit: 10) { id balance firstDepositDay }
}
```

Read the live share price with `getRate()` on the accountant.

### Make it live

There's no `end_block` in `config.yaml`, so once the backfill is done the indexer keeps following Monad's head and your numbers keep growing.

### Deploy it

Push your fork to GitHub and deploy it on [Envio Cloud](https://docs.envio.dev/docs/HyperIndex/hosted-service) to get a hosted GraphQL endpoint. Indexers on Envio Cloud don't need your own API token. The [deployment guide](https://docs.envio.dev/docs/HyperIndex/hosted-service-deployment) covers connecting the repo.

## Check the numbers yourself

[`scripts/full-run.test.ts`](scripts/full-run.test.ts) runs the same handler over the vault's whole history in memory, with no Postgres, and writes `out/summary.json` and `out/holders.json`. The copies in this repo are the run behind the numbers above.

```bash
pnpm full-run                      # stops at block 122,726,000
END_BLOCK=123000000 pnpm full-run  # or pick your own block
```

The sum of every balance in `out/holders.json` is 60,044,007.367598 shares. Read the vault's `totalSupply()` from Monad's public RPC at the same block and you get the same figure.

```bash
curl -s https://rpc.monad.xyz -H 'content-type: application/json' -d '{"jsonrpc":"2.0","id":1,"method":"eth_call","params":[{"to":"0x1C8a336051D2024E318A229d01F9F6CF96efD316","data":"0x18160ddd"},"0x750a670"]}'
# result 0x369c15bae3ae = 60044007367598 (6 decimals)
```

We also checked the rest outside the repo:

- A separate HyperSync script pulled all 403,875 vault `Transfer` logs to block 122,726,000 and rebuilt every balance. Its 69,374 non-zero balances match the indexer's wallet by wallet.
- `balanceOf` over public RPC at the same block for 20 randomly picked holders matched to the last unit, and wallets the rebuild shows as fully withdrawn returned 0.
- For 15 Nov, every vault mint pulled from public RPC with `eth_getLogs` (blocks 122,308,133 to 122,592,638) gave the same 1,893 mints to the same wallets as HyperSync, and the same 788 first-time wallets as the indexer.
- Aave's own API reported a 56.01M mUSD supply position for the vault, in line with the RPC read.

## Build your own with an AI agent

This repo doesn't ship Envio's agent skills yet. To add them, run `pnpx envio init` in an empty folder and copy the `.claude/skills/` folder it creates into this repo, or follow [Quickstart with AI](https://docs.envio.dev/docs/HyperIndex/quickstart-with-ai). Then paste a prompt like this into Claude Code or Cursor from the repo root.

```text
This is an Envio HyperIndex indexer on Monad (chain 143) for the Veda BoringVault
behind MetaMask's Money Account. Read README.md, config.yaml, schema.graphql,
src/handlers/MoneyAccountVault.ts, scripts/full-run.test.ts and any skills in
.claude/skills, then https://docs.envio.dev/docs/HyperIndex/testing.

Add a second vault: <VAULT ADDRESS> on <CHAIN>, from its first Transfer block.
Key Totals, Holder and DailyStat by vault so the numbers stay separate. Add a
weekly rollup of new wallets next to the daily one. Add a "test" script and unit
tests using createTestIndexer that cover a mint, a burn and a transfer between
holders, run `pnpm codegen` and `pnpm test` until they pass, then run `pnpm dev`
and show me the GraphQL query for new wallets per week per vault.
```

Swap in any other Veda BoringVault or ERC-4626 vault. Both mint and burn their shares with a standard ERC-20 `Transfer`, so the handler carries over as it is.

## Links

- [HyperIndex overview](https://docs.envio.dev/docs/HyperIndex/overview)
- [HyperSync supported networks](https://docs.envio.dev/docs/HyperSync/hypersync-supported-networks)
- [Testing HyperIndex indexers](https://docs.envio.dev/docs/HyperIndex/testing)
- [Envio Cloud](https://docs.envio.dev/docs/HyperIndex/hosted-service)
- [Quickstart with AI](https://docs.envio.dev/docs/HyperIndex/quickstart-with-ai)
- [MetaMask Money Account](https://support.metamask.io/trade/money-account/)
- [Introducing MetaMask Money Account](https://metamask.com/news/introducing-metamask-money-account)
- [Steakhouse on the Money Account](https://kitchen.steakhouse.financial/p/the-metamask-money-account)
- [Veda docs](https://docs.veda.tech)
- [Aave v3 docs](https://aave.com/docs)
- [Aave address book for Monad](https://github.com/bgd-labs/aave-address-book/blob/main/src/AaveV3Monad.sol)
- [Monad docs](https://docs.monad.xyz)
- [Monad protocols registry entry](https://github.com/monad-crypto/protocols/blob/main/mainnet/metamask.jsonc)
- [Envio Discord](https://discord.gg/envio)
