# Auto_install script
```python
wget -O lava https://raw.githubusercontent.com/Node-max/Testnet/main/Lava/Node_installation_guide/Lava && chmod +x lava && ./lava
```

# Manual installation

### Preparing the server

```python
sudo apt update && sudo apt upgrade -y
sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make ncdu gcc git jq chrony liblz4-tool -y
```

## GO 1.19

```python
ver="1.19" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
source $HOME/.bash_profile && \
go version
```

# Build 28.03.23
```python
cd $HOME
git clone https://github.com/lavanet/lava
cd lava
git fetch --all
git checkout v0.8.1
make install
```
*******UPDATE******* 28.03.23

```python
cd $HOME/lava
git fetch --all
git checkout v0.8.1
make install
lavad version --long | head
sudo systemctl restart lavad && sudo journalctl -u lavad -f -o cat
```

`lavad version --long | head`
- version: 0.8.1
- commit: 910bfdf6fca1ba4030f4587c3f5af3382a8b5238

```python
lavad init STAVRguide --chain-id lava-testnet-1
lavad config chain-id lava-testnet-1
```    

## Create/recover wallet
```python
lavad keys add <walletname>
      OR
lavad keys add <walletname> --recover
```

## Download Genesis
```python
wget -O ~/.lava/config/genesis.json https://raw.githubusercontent.com/Node-max/Testnet/main/Lava/Node_installation_guide/genesis.json
```
`sha256sum $HOME/.lava/config/genesis.json`
+ 72170a8a7314cb79bc57a60c1b920e26457769667ce5c2ff0595b342c0080d78

## Set up the minimum gas price and Peers/Seeds/Filter peers/MaxPeers
```python
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0ulava\"/" $HOME/.lava/config/app.toml
sed -i -e "s/^filter_peers *=.*/filter_peers = \"true\"/" $HOME/.lava/config/config.toml
external_address=$(wget -qO- eth0.me) 
sed -i.bak -e "s/^external_address *=.*/external_address = \"$external_address:26656\"/" $HOME/.lava/config/config.toml
peers="3a445bfdbe2d0c8ee82461633aa3af31bc2b4dc0@prod-pnet-seed-node.lavanet.xyz:26656,e593c7a9ca61f5616119d6beb5bd8ef5dd28d62d@prod-pnet-seed-node2.lavanet.xyz:26656"
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$peers\"/" $HOME/.lava/config/config.toml
seeds=""
sed -i.bak -e "s/^seeds =.*/seeds = \"$seeds\"/" $HOME/.lava/config/config.toml
sed -i 's/create_empty_blocks = .*/create_empty_blocks = true/g' ~/.lava/config/config.toml
sed -i 's/create_empty_blocks_interval = ".*s"/create_empty_blocks_interval = "60s"/g' ~/.lava/config/config.toml
sed -i 's/timeout_propose = ".*s"/timeout_propose = "60s"/g' ~/.lava/config/config.toml
sed -i 's/timeout_commit = ".*s"/timeout_commit = "60s"/g' ~/.lava/config/config.toml
sed -i 's/timeout_broadcast_tx_commit = ".*s"/timeout_broadcast_tx_commit = "601s"/g' ~/.lava/config/config.toml
sed -i 's/max_num_inbound_peers =.*/max_num_inbound_peers = 50/g' $HOME/.lava/config/config.toml
sed -i 's/max_num_outbound_peers =.*/max_num_outbound_peers = 50/g' $HOME/.lava/config/config.toml

```
### Pruning (optional)
```python
pruning="custom" && \
pruning_keep_recent="100" && \
pruning_keep_every="0" && \
pruning_interval="10" && \
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" ~/.lava/config/app.toml && \
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" ~/.lava/config/app.toml && \
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" ~/.lava/config/app.toml && \
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" ~/.lava/config/app.toml
```
### Indexer (optional) 
```python
indexer="null" && \
sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.lava/config/config.toml
```

## Download addrbook
```python
wget -O $HOME/.lava/config/addrbook.json "https://raw.githubusercontent.com/Node-max/Testnet/main/Lava/Node_installation_guide/addrbook.json"
```
## StateSync
```python
SNAP_RPC=https://rpc.lava.max-node.xyz:443
peers="69549b8047c98609d9935415aec7999386eb6d07@5.182.33.99:44656"
sed -i.bak -e  "s/^persistent_peers *=.*/persistent_peers = \"$peers\"/" ~/.lava/config/config.toml
LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height); \
BLOCK_HEIGHT=$((LATEST_HEIGHT - 500)); \
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)

echo $LATEST_HEIGHT $BLOCK_HEIGHT $TRUST_HASH

sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"| ; \
s|^(seeds[[:space:]]+=[[:space:]]+).*$|\1\"\"|" $HOME/.lava/config/config.toml
lavad tendermint unsafe-reset-all --home /root/.lava --keep-addr-book
sed -i -e "s/^snapshot-interval *=.*/snapshot-interval = \"1500\"/" $HOME/.lava/config/app.toml
systemctl restart lavad && journalctl -u lavad -f -o cat
```

## SnapShot (~0.1 GB) updated every 5 hours  
```python
SOOOOOOOOOON
```

# Create a service file
```python
sudo tee /etc/systemd/system/lavad.service > /dev/null <<EOF
[Unit]
Description=lava
After=network-online.target

[Service]
User=$USER
ExecStart=$(which lavad) start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

## Start
```python
sudo systemctl daemon-reload
sudo systemctl enable lavad
sudo systemctl restart lavad && sudo journalctl -u lavad -f -o cat
```

## Delete node
```bash
sudo systemctl stop lavad && \
sudo systemctl disable lavad && \
rm /etc/systemd/system/lavad.service && \
sudo systemctl daemon-reload && \
cd $HOME && \
rm -rf GHFkqmTzpdNLDd6T && \
rm -rf .lava && \
rm -rf $(which lavad)
```
#
### Sync Info
```python
lavad status 2>&1 | jq .SyncInfo
```
### NodeINfo
```python
lavad status 2>&1 | jq .NodeInfo
```
### Check node logs
```python
sudo journalctl -u lavad -f -o cat
```
### Check Balance
```python
lavad query bank balances lava@1a2pvmd..............rflml4hg2a0x3ky
```
