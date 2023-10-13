## Summary

This repositrory contains code and instructions for submitting a [`swingset.CoreEval`](https://docs.agoric.com/guides/coreeval/) proposal to the [Agoric](https://github.com/Agoric/agoric-sdk/) blockchain. CoreEval is a special type of governance proposal that executes code after a passing vote. 

Specifically, this is a proposal for adding `stATOM` as a collateral type to [Inter Protocol](https://github.com/Agoric/agoric-sdk/tree/master/packages/inter-protocol) Vaults. See the [community discussion](https://community.agoric.com/t/onboard-statom-as-collateral/410) for more details.

This proposal can easily be adapted for other collerateral types, and should serve as a reference for future proposals. You can see a demo and discussion of this code recorded during developer [office hours](https://github.com/Agoric/agoric-sdk/discussions/8240).

## Preqrequisites

### 1. Use Proposal Builder to generate Core Eval files

See [builders/scripts/inter-protocol/add-STARS.js](https://github.com/Agoric/agoric-sdk/blob/4df6e758409e9196fdf0381904a4caccec99545d/packages/builders/scripts/inter-protocol/add-STARS.js) and this npm [script](https://github.com/Agoric/agoric-sdk/blob/02e420b5f72ef6e47cdb7b1cc4a2b8403955fc92/packages/builders/package.json#L10) for more details.

I have checked in the generated files here for reference. The files were generated from [this commit hash]https://github.com/Agoric/agoric-sdk/tree/4df6e758409e9196fdf0381904a4caccec99545d), using the added `yarn build:add-stATOM-proposal` script.


Alternatively, you can edit the files provided here directly, adjusting the sections highlighted:

 - [add-stATOM.js#L9-L18](https://github.com/0xpatrickdev/agoric-vault-collateral-proposal/blob/main/add-stATOM.js#L9-L18)
 - [add-stATOM-oracles.js#L8-L45](https://github.com/0xpatrickdev/agoric-vault-collateral-proposal/blob/main/add-stATOM-oracles.js#L8-L45)

### 2. Setup wallet

One wallet is needed for submitting governance proposals, and two* wallets are required for submitting oracle prices.

```zsh
# list current keys
agd keys list
# add a new key, with auto-generated mnemonic
agd keys add [dev-local]
# add a new key, with custom mnemonic
agd keys add [dev-local] --interactive
# prints address
agd keys show [dev-local] -a
```

* two if `minSubmissionCount` in `decentral-devnet-config.json` is set to `2`

### 2.a. Setting up a hardware wallet
 
Ledger hardware wallet signing is supported by agd, and can be set up as follows:

```zsh
agd keys add [my-ledger] --ledger --coin-type=118 [--account=0] [--index=0]
# use the hw key to sign and submit a tx
agd tx swingset [...] --from=[my-ledger] --sign-mode=amino-json
```

### 3. Request faucet funds + Provision Wallet

Setup a smart-wallet and request funds here: https://devnet.faucet.agoric.net/, using the address from the previous step.

In a local environment, the following `cosmic-swingset` Makefile commands will help achieve the same:
```zsh
cd ~/agoric-sdk/packages/cosmic-swingset
# fund provision pool
make fund-provision-pool
# create a smart wallet
ACCT_ADDR=your-address make provision-acct
# add faucet tokens to wallet
ACCT_ADDR=your-address FUNDS=9999000000ubld,9999000000uist make fund-acct
```

## Proposal Steps

### 1. Deploy bundles
_Before deploying bundles, you will want to query the chain [1.a.](#1a-verify-bundle-deployment) to see if they are already published. If they are, which is likely the case for a Vault Collateral Proposal, you can avoid the gas cost of deploying them again and skip this step._

```zsh
NODE=https://devnet.rpc.agoric.net:443
WALLET=dev-local
CHAIN_ID=agoricdev-20
B1=b1-903e41a7c448a41b456298404a1c32c69302574209c6a5228723ed19e2dd99f2a693641196445bc27a90e19e1dfadfe6b3d9c9a93f080ffa33a70908e5af4fff.json
B2=b1-80e6fe68b299c82c2d26802c312bc37966a559f7b28f87d058887a79a9db48ad97da2240e71e3f98986071da8fc3c5d02358bec577b17a89cee2b1cb3cd23958.json


cd bundles
agd tx swingset install-bundle @$B1 --node $NODE --from $WALLET --chain-id $CHAIN_ID --gas=auto --gas-adjustment=1.2 -y -b block
agd tx swingset install-bundle @$B2 --node $NODE --from $WALLET --chain-id $CHAIN_ID --gas=auto --gas-adjustment=1.2 -y -b block
```
_Alternatively, the `deploy-bundles.sh` script can be used to ensure only un-published bundles are submitted._


### 1.a. Verify Bundle Deployment

```zsh
# returns a global list of deployed bundles
agd query vstorage data bundles --node $NODE --chain-id $CHAIN_ID --output json
# returns a formattted list of deployed bundles
agd query vstorage data bundles --node $NODE --chain-id $CHAIN_ID --output json | \
jq -r '.value | fromjson | .values | map(fromjson) | .[-1] | .body[1:] | fromjson'
```

### 2. Submit Governance Proposal

```zsh
NODE=https://devnet.rpc.agoric.net:443
WALLET=dev-local
CHAIN_ID=agoricdev-21 // see https://devnet.agoric.net/ for latest chain id
agd tx gov submit-proposal swingset-core-eval \
  add-stATOM-permit.json add-stATOM.js \
  add-stATOM-oracles-permit.json add-stATOM-oracles.js \
  --title="Enable stATOM Vault" --description="Evaluate add-stATOM.js add-stATOM-oracles.js" --deposit=1000000ubld \
  --gas=auto --gas-adjustment=1.2 \
  --node $NODE --from $WALLET --chain-id $CHAIN_ID
```

### 3. Query Gov Proposals

```zsh
NODE=https://devnet.rpc.agoric.net:443
WALLET=dev-local
CHAIN_ID=agoricdev-20
agd query gov proposals --node $NODE --chain-id $CHAIN_ID --output json | \
  jq -c '.proposals[] | [.proposal_id,.voting_end_time,.status]';
```

### 4. Vote on Gov Proposal

```zsh
NODE=https://devnet.rpc.agoric.net:443
WALLET=dev-local
CHAIN_ID=agoricdev-20
agd tx gov vote 1 yes --node $NODE --from $WALLET --chain-id $CHAIN_ID
```


<img alt="gov proposal on explorer" width="600px" src="docs/proposal.png" />

## Oracle Steps

_If you are testing on a local chain, you will need to send oracle prices in order for the vault manager to finish initializing._

Before, ensure at least two addresses you control are listed in `oracleAddresses`. You may also want to adjust `minSubmissionCount` from `3` to `2` in `decentral-devnet-config.json`.

```zsh
cd ~/agoric-sdk
WALLET=dev-local
NODE=http://localhost:26657
WALLET=dev-local
WALLET_2=dev-local-2
CHAIN_ID=agoriclocal
alias oracle="yarn run --silent agops oracle"

# accept the offer to submit a price
oracle accept --offerId 1 --pair stATOM.USD > offer-1-w1.json
agoric wallet send --from $WALLET --offer offer-1-w1.json

# push a price
oracle pushPriceRound --price 10 --roundId 1 --oracleAdminAcceptOfferId 1 > price-offer-1-w1.json
agoric wallet send --from $WALLET --offer price-offer-1-w1.json

# verify price feed
agoric follow :published.priceFeed.stATOM-USD_price_feed

# submit a price from wallet 2
AGORIC_NET=devnet oracle accept --offerId 1 --pair stATOM.USD > offer-1-w2.json
AGORIC_NET=devnet agoric wallet send --from $WALLET_2 --offer offer-1-w2.json
AGORIC_NET=devnet oracle pushPriceRound --price 10 --roundId 1 --oracleAdminAcceptOfferId 1 > price-offer-1-w2.json
AGORIC_NET=devnet agoric wallet send --from $WALLET_2 --offer price-offer-1-w2.json
```

When using the agops cli, the network can be specified by prefacing commands with `AGORIC_NET=devnet`.

## Open a Vault with agops cli
```zsh
alias vaults="yarn run --silent agops vaults"
AGORIC_NET=devnet vaults open --wantMinted 5.00 --giveCollateral 9.0 --collateralBrand stATOM > open-vault-offer.json
AGORIC_NET=devnet agoric wallet send --from $WALLET --offer open-vault-offer.json
```


## UI Validation

_A link to an endorsed UI can be found on https://devnet.agoric.net/._

<img alt="stATOM vault in dapp ui" width="600px" src="docs/vault-ui.png" />


## REPL Validation

```js
// looking your wallet, so you can request faucet funds
E(home.myAddressNameAdmin).getMyAddress()

// request a loan for 5m uist (the minimum)
E(E(home.agoricNames).lookup('issuer', 'IST')).getBrand()
istBrand = history[n]
fiveMillionUistAmount = { brand: istBrand, value: 5_000_000n}

// find the vault factory instance
E(home.agoricNames).lookup('instance', 'VaultFactory')
vfi = history[n]

// open an ATOM vault
E(E(home.agoricNames).lookup('issuer', 'ATOM')).getBrand()
atomBrand = history[n]
atomPurse = E(home.wallet).getPurse("ATOM")
oneMillionUatomAmount = { brand: atomBrand, value: 1_000_000n}
proposal = { give: { Collateral: oneMillionUatomAmount }, want: { Minted: fiveMillionUistAmount } }
E(atomPurse).withdraw(oneMillionUatomAmount)
pmt = { Collateral: history[n] }
E(E(E(home.zoe).getPublicFacet(vfi)).getCollateralManager(atomBrand)).makeVaultInvitation()
inv = history[n]
E(home.zoe).offer(inv, proposal, pmt)
E(history[n]).getOfferResult() // should provide a VaultSeatKit, or show an error

// open an stATOM vault
E(E(home.agoricNames).lookup('issuer', 'stATOM')).getBrand()
stAtomBrand = history[n]
stAtomPurse = E(home.wallet).getPurse("stATOM")
oneMillionUstAtomAmount = { brand: stAtomBrand, value: 1_000_000n}
proposal = { give: { Collateral: oneMillionUstAtomAmount }, want: { Minted: fiveMillionUistAmount } }
E(stAtomPurse).withdraw(oneMillionUstAtomAmount)
pmt = { Collateral: history[n] }
E(E(E(home.zoe).getPublicFacet(vfi)).getCollateralManager(stAtomBrand)).makeVaultInvitation()
inv = history[n]
E(home.zoe).offer(inv, proposal, pmt)
E(history[n]).getOfferResult() // should provide a VaultSeatKit, or show an error
```

## REPL Oracle Validation

```js
E(home.agoricNames).lookup('oracleBrand', 'stATOM')
E(home.agoricNames).lookup('oracleBrand', 'USD')
stAtomOracleBrand = history[n]
E(home.priceAuthority).makeQuoteNotifier({ value: 10n * 10n ** 6n, brand: stAtomOracleBrand }, usdOracleBrand)
E(qn).getUpdateSince()
```
