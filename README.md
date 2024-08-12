# Guide-0g-Node

# Hardware Requirement
```
- CPU: 4 cores
- Memory: 8 GB RAM
- Disk: 500 GB SSD
```

## Install Required Packages

```bash
sudo apt update && \
sudo apt install curl git jq build-essential gcc unzip wget lz4 -y
```

## Install Go

```bash
cd $HOME && \
ver="1.21.3" && \
wget "<https://golang.org/dl/go$ver.linux-amd64.tar.gz>" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile && \
source ~/.bash_profile
```

## Build evmosd Binary

```bash
wget https://rpc-zero-gravity-testnet.warlord44.host/evmosd
chmod +x ./evmosd
mv ./evmosd /usr/local/bin/
```

## Set up Variables

You can change MONIKERNAME however you want

```bash
echo 'export MONIKER="MONIKERNAME"' >> ~/.bash_profile
echo 'export CHAIN_ID="zgtendermint_9000-1"' >> ~/.bash_profile
echo 'export WALLET_NAME="wallet"' >> ~/.bash_profile
echo 'export RPC_PORT="26657"' >> ~/.bash_profile
source $HOME/.bash_profile
```


## Initialize the Node

```bash
cd $HOME
evmosd init $MONIKER --chain-id $CHAIN_ID
evmosd config chain-id $CHAIN_ID
evmosd config node tcp://localhost:$RPC_PORT
evmosd config keyring-backend os
```

## Download genesis.json

```bash
wget https://rpc-zero-gravity-testnet.warlord44.host/genesis.json -O $HOME/.evmosd/config/genesis.json
```

## Add Seeds and Peers to the config.toml

```bash
PEERS="1248487ea585730cdf5d3c32e0c2a43ad0cda973@peer-zero-gravity-testnet.warlord44.host:26326" && \
SEEDS="8c01665f88896bca44e8902a30e4278bed08033f@54.241.167.190:26656,b288e8b37f4b0dbd9a03e8ce926cd9c801aacf27@54.176.175.48:26656,8e20e8e88d504e67c7a3a58c2ea31d965aa2a890@54.193.250.204:26656,e50ac888b35175bfd4f999697bdeb5b7b52bfc06@54.215.187.94:26656" && \
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.evmosd/config/config.toml
```

## Change Ports | Optional

```bash
EXTERNAL_IP=$(wget -qO- eth0.me) \
PROXY_APP_PORT=26658 \
P2P_PORT=26656 \
PPROF_PORT=6060 \
API_PORT=1317 \
GRPC_PORT=9090 \
GRPC_WEB_PORT=9091
```

```bash
sed -i \
    -e "s/\(proxy_app = \"tcp:\/\/\)\([^:]*\):\([0-9]*\).*/\1\2:$PROXY_APP_PORT\"/" \
    -e "s/\(laddr = \"tcp:\/\/\)\([^:]*\):\([0-9]*\).*/\1\2:$RPC_PORT\"/" \
    -e "s/\(pprof_laddr = \"\)\([^:]*\):\([0-9]*\).*/\1localhost:$PPROF_PORT\"/" \
    -e "/\[p2p\]/,/^\[/{s/\(laddr = \"tcp:\/\/\)\([^:]*\):\([0-9]*\).*/\1\2:$P2P_PORT\"/}" \
    -e "/\[p2p\]/,/^\[/{s/\(external_address = \"\)\([^:]*\):\([0-9]*\).*/\1${EXTERNAL_IP}:$P2P_PORT\"/; t; s/\(external_address = \"\).*/\1${EXTERNAL_IP}:$P2P_PORT\"/}" \
    $HOME/.evmosd/config/config.toml
```

```bash
sed -i \
    -e "/\[api\]/,/^\[/{s/\(address = \"tcp:\/\/\)\([^:]*\):\([0-9]*\)\(\".*\)/\1\2:$API_PORT\4/}" \
    -e "/\[grpc\]/,/^\[/{s/\(address = \"\)\([^:]*\):\([0-9]*\)\(\".*\)/\1\2:$GRPC_PORT\4/}" \
    -e "/\[grpc-web\]/,/^\[/{s/\(address = \"\)\([^:]*\):\([0-9]*\)\(\".*\)/\1\2:$GRPC_WEB_PORT\4/}" $HOME/.evmosd/config/app.toml
```

## Configure Prunning to Save Storage | Optional

```bash
sed -i.bak -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.evmosd/config/app.toml
sed -i.bak -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.evmosd/config/app.toml
sed -i.bak -e "s/^pruning-interval *=.*/pruning-interval = \"10\"/" $HOME/.evmosd/config/app.toml
```

## Set Min Gas Price

```bash
sed -i "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.00252aevmos\"/" $HOME/.evmosd/config/app.toml
```

## Enable Indexer | Optional

```bash
sed -i "s/^indexer *=.*/indexer = \"kv\"/" $HOME/.evmosd/config/config.toml
```

## Create a Service File

```bash
sudo tee /etc/systemd/system/ogd.service > /dev/null << EOF
[Unit]
Description=OG Node
After=network.target
[Service]
User=$USER
Type=simple
ExecStart=$(which evmosd) start --home $HOME/.evmosd
Restart=on-failure
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

## Start the Node

```bash
sudo systemctl daemon-reload && \
sudo systemctl enable ogd && \
sudo systemctl restart ogd && \
sudo journalctl -u ogd -f -o cat
```

## Stop the Node

```bash
sudo systemctl stop ogd
```

## Download Snapshot

```bash
wget https://rpc-zero-gravity-testnet.warlord44.host/latest_snapshot.tar.lz4
```

## Backup priv_validator_state.json

```bash
cp $HOME/.evmosd/data/priv_validator_state.json $HOME/.evmosd/priv_validator_state.json.backup
```

## Reset Database

```bash
evmosd tendermint unsafe-reset-all --home $HOME/.evmosd --keep-addr-book
```

## Extract Files from the Arvhive

```bash
lz4 -d -c ./latest_snapshot.tar.lz4 | tar -xf - -C $HOME/.evmosd
```

## Move priv_validator_state.json Back

```bash
mv $HOME/.evmosd/priv_validator_state.json.backup $HOME/.evmosd/data/priv_validator_state.json
```

## Restart the Node

```bash
sudo systemctl restart ogd && sudo journalctl -u ogd -f -o cat
```

## Create a Wallet for Your Validator

```bash
evmosd keys add $WALLET_NAME
```

**NOTE!**

**Save seed pharse. If you don't save the seed pharse you can't get it back**

If you want to recover wallet use command:

```bash
evmosd keys add $WALLET_NAME --recover
```

## Extract the HEX Address to Request Some Tokens from the Faucet

```bash
echo "0x$(evmosd debug addr $(evmosd keys show $WALLET_NAME -a) | grep hex | awk '{print $3}')"
```

This command will appear an address to use for faucet tokens

Can extract the private keys in address above, use command:

```bash
evmosd keys unsafe-export-eth-key $WALLET_NAME
```

you can add account with private keys in metamask, account use faucet token if you want

**NOTE!**

**Save private keys add to metamask to faucet token**

## Faucet
Link: https://faucet.0g.ai/

**NOTE**

**To join the active validators set you need at least *1000000000000000000 aevmos***


## Check Wallet Balances

```bash
evmosd q bank balances $(evmosd keys show $WALLET_NAME -a)
```

## Check SyncInfo

```
evmosd status | jq .SyncInfo.catching_up
```

If `catching_up` is `false` you can create validator

## Create a Validator

```bash
evmosd tx staking create-validator \
--amount=10000000000000000aevmos \
--pubkey=$(evmosd tendermint show-validator) \
--moniker=$MONIKER \
--chain-id=$CHAIN_ID \
--commission-rate=0.05 \
--commission-max-rate=0.10 \
--commission-max-change-rate=0.01 \
--min-self-delegation=1 \
--from=$WALLET_NAME \
--gas=500000 --gas-prices=99999aevmos \
-y
```

## Backup Validator Keys

```bash
cp $HOME/.evmosd/data/priv_validator_state.json $HOME/.evmosd/priv_validator_state.json.backup
```

## Check Validator

```bash
evmosd q staking validator $(evmosd keys show $WALLET_NAME --bech val -a)
```
