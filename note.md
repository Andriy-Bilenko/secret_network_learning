# setup

```sh
rustc --version
cargo --version
rustup target list --installed
# if wasm32 is not listed above, run this
rustup target add wasm32-unknown-unknown
# get the cargo generate
cargo install cargo-generate --features vendored-openssl
```

## get the template

```bash
# pull the template
cargo generate --git https://github.com/scrtlabs/secret-template.git --name my-counter-contract
```

## get local secret (in docker)

```bash
# run in docker, local secret blockchain
docker run -it -p 9091:9091 -p 26657:26657 -p 1317:1317 -p 5000:5000 \
  --name localsecret ghcr.io/scrtlabs/localsecret:latest
```

## get secretcli (to interact with blockchain (local or remote))

```bash
# install secret cli
# SecretCLI is a command-line tool that lets us interact with the Secret Network blockchain. It is used to send and query data as well as manage user keys and wallets.
wget https://github.com/scrtlabs/SecretNetwork/releases/latest/download/secretcli-Linux
chmod +x secretcli-Linux
sudo mv secretcli-Linux /usr/local/bin/secretcli
```

### config secretcli for local secret

```bash
# configuring secretcli for localsecret
secretcli config node http://localhost:26657
secretcli config chain-id secretdev-1
secretcli config keyring-backend test
secretcli config output json
```

# compile code

```bash
# outputs `contract.wasm` and `contract.wasm.gz` in root dir
make build
```

# optimise code

```bash
# gives out optimised `contract.wasm.gz`
# NOTE: takes long after `Updating crates.io index` so be patient, this is OK
docker run --rm -v "$(pwd)":/contract \
  --mount type=volume,source="$(basename "$(pwd)")_cache",target=/code/target \
  --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
  enigmampc/secret-contract-optimizer
```

# deploying to blockchain

uploading wasm to blockchain is called **storing the contract code**, for that we need a wallet to pay fees

## create wallet

```bash
secretcli keys add myWallet
```

it should give out:

```json
{
	"name": "myWallet",
	"type": "local",
	"address": "secret1cwcd085acvz84z7kczfzw0pysmpq85cdlarcer",
	"pubkey": "{\"@type\":\"/cosmos.crypto.secp256k1.PubKey\",\"key\":\"AqjW6iv9pSCfDydKFiznqqQTNr8cwpbRmf8abJcslE0A\"}",
	"mnemonic": "make witness citizen height escape shallow answer blood table omit embrace please iron draw priority jelly hungry success consider select fatal innocent chaos snap"
}
```

```bash
# get balance
secretcli query bank balances "secret1cwcd085acvz84z7kczfzw0pysmpq85cdlarcer"
# get 1_000_000_000 scrt from faucet
curl "http://localhost:5000/faucet?address=secret1cwcd085acvz84z7kczfzw0pysmpq85cdlarcer"
```

## upload contract

```bash
# name is wallet address
secretcli tx compute store contract.wasm.gz --gas 5000000 --from <name> --chain-id secretdev-1
# so it is like
secretcli tx compute store contract.wasm.gz --gas 5000000 --from secret1cwcd085acvz84z7kczfzw0pysmpq85cdlarcer --chain-id secretdev-1

# query the results uploaded
secretcli query compute list-code
```

## instantiate the contract

```bash
# 1 is code id from query above
# {"count": 1} is the instantiation message. Here we instantiate a starting count of 1, but you can make it any i32 you want
# --label is a mandatory field that gives the contract a unique meaningful identifier
secretcli tx compute instantiate 1 '{"count": 1}' --from <name> --label counterContract -y
# so it is like
secretcli tx compute instantiate 1 '{"count": 1}' --from secret1cwcd085acvz84z7kczfzw0pysmpq85cdlarcer --label counterContract -y

# confirm instantiation
secretcli query compute list-contract-by-code 1
```

the result should be

```json
[
	{
		"contract_address": "secret1fguk8uxavvx9frgmaygm6qw0m4nzpfgw829spr",
		"code_id": 1,
		"creator": "secret1cwcd085acvz84z7kczfzw0pysmpq85cdlarcer",
		"label": "counterContract",
		"admin_proof": "5V/z4GK3qpZvfoMHx6DMEpFWTj9LkDx7WDMD/l1Jh+0="
	}
]
```
