# Manual installation
### Setup validator name
```python
MONIKER="YOUR_MONIKER_GOES_HERE"
```
### Preparing the server
```python
sudo apt -q update
sudo apt -qy install curl git jq lz4 build-essential
sudo apt -qy upgrade
```
### GO 1.19
```python
sudo rm -rf /usr/local/go
curl -Ls https://go.dev/dl/go1.20.2.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
eval $(echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/golang.sh)
eval $(echo 'export PATH=$PATH:$HOME/go/bin' | tee -a $HOME/.profile)
```
# Node installation
### Clone project repository
```python
cd $HOME
rm -rf nibiru
git clone https://github.com/NibiruChain/nibiru.git
cd nibiru
git checkout v0.19.2
# Build binaries
make build
```
### Prepare binaries for Cosmovisor
```python
mkdir -p $HOME/.nibid/cosmovisor/genesis/bin
mv build/nibid $HOME/.nibid/cosmovisor/genesis/bin/
rm -rf build
``` 
### Create application symlinks
```python
ln -s $HOME/.nibid/cosmovisor/genesis $HOME/.nibid/cosmovisor/current
sudo ln -s $HOME/.nibid/cosmovisor/current/bin/nibid /usr/local/bin/nibid
```
# Install Cosmovisor and create a service
### Download and install Cosmovisor
```python
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.4.0
```
### Create service
```python
sudo tee /etc/systemd/system/nibid.service > /dev/null << EOF
[Unit]
Description=nibiru-testnet node service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.nibid"
Environment="DAEMON_NAME=nibid"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.nibid/cosmovisor/current/bin"

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable nibid
```
# Initialize the node
### Set node configuration
```python
nibid config chain-id nibiru-itn-1
nibid config keyring-backend test
nibid config node tcp://localhost:39657
```
### Initialize the node
```python
nibid init $MONIKER --chain-id nibiru-itn-1
```
### Download genesis and addrbook
```pythom
curl -Ls https://snapshot.max-node.xyz/nibiru/genesis.json > $HOME/.nibid/config/genesis.json
curl -Ls https://snapshot.max-node.xyz/nibiru/addrbook.json > $HOME/.nibid/config/addrbook.json
```
### Add seeds
```python
sed -i -e "s|^seeds *=.*|seeds = \"3f472746f46493309650e5a033076689996c8881@rpc.nibiru.max-node.xyz:39657\"|" $HOME/.nibid/config/config.toml
```
### Set minimum gas price
```python
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.025unibi\"|" $HOME/.nibid/config/app.toml```
### Set pruning
```python
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.nibid/config/app.toml
```
### Set custom ports
```python
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:39658\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:39657\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:39060\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:39656\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":39660\"%" $HOME/.nibid/config/config.toml
sed -i -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:39317\"%; s%^address = \":8080\"%address = \":39080\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:39090\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:39091\"%; s%^address = \"0.0.0.0:8545\"%address = \"0.0.0.0:39545\"%; s%^ws-address = \"0.0.0.0:8546\"%ws-address = \"0.0.0.0:39546\"%" $HOME/.nibid/config/app.toml
```
### Download latest chain snapshot
```python
curl -L https://snapshot.max-node.xyz/nibiru/nibiru-itn-1_latest.tar.lz4 | tar -Ilz4 -xf - -C $HOME/.nibid
[[ -f $HOME/.nibid/data/upgrade-info.json ]] && cp $HOME/.nibid/data/upgrade-info.json $HOME/.nibid/cosmovisor/genesis/upgrade-info.json
```
### Start service and check the logs
```python
sudo systemctl start nibid && sudo journalctl -u nibid -f --no-hostname -o cat
```


# Set up a pricefeeder
## To run pricefeeder you validator should be in active set. Otherwise price feeder will not vote on periods.
### Install the pricefeeder binary
```python
curl -s https://get.nibiru.fi/pricefeeder! | bash
```
### Create new wallet for pricefeeder and save 24 word mnemonic phrase
```python
nibid keys add pricefeeder-wallet
```
## Top up the pricefeeder-wallet going to Nibiru Discord [#faucet](https://discord.gg/nibiru) channel.
## In order to make pricefeeder work, it needs some testnet tokens to pay for transaction fees
## Export pricefeeder mnemonic into environment variable
```python
export FEEDER_MNEMONIC="my pricefeeder 24 word mnemonic phrase goes here ..."
```
### Setup the systemd service
```python
export CHAIN_ID="nibiru-itn-1"
export GRPC_ENDPOINT="localhost:39090"
export WEBSOCKET_ENDPOINT="ws://localhost:39657/websocket"
export EXCHANGE_SYMBOLS_MAP='{ "bitfinex": { "ubtc:uusd": "tBTCUSD", "ueth:uusd": "tETHUSD", "uusdt:uusd": "tUSTUSD" }, "binance": { "ubtc:uusd": "BTCUSD", "ueth:uusd": "ETHUSD", "uusdt:uusd": "USDTUSD", "uusdc:uusd": "USDCUSD", "uatom:uusd": "ATOMUSD", "ubnb:uusd": "BNBUSD", "uavax:uusd": "AVAXUSD", "usol:uusd": "SOLUSD", "uada:uusd": "ADAUSD", "ubtc:unusd": "BTCUSD", "ueth:unusd": "ETHUSD", "uusdt:unusd": "USDTUSD", "uusdc:unusd": "USDCUSD", "uatom:unusd": "ATOMUSD", "ubnb:unusd": "BNBUSD", "uavax:unusd": "AVAXUSD", "usol:unusd": "SOLUSD", "uada:unusd": "ADAUSD" } }'
export VALIDATOR_ADDRESS=$(nibid keys show wallet --bech val -a)

sudo tee /etc/systemd/system/pricefeeder.service<<EOF
[Unit]
Description=Nibiru Pricefeeder
Requires=network-online.target
After=network-online.target

[Service]
Type=exec
User=$USER
Group=$USER
ExecStart=/usr/local/bin/pricefeeder
Restart=on-failure
ExecReload=/bin/kill -HUP $MAINPID
KillSignal=SIGTERM
PermissionsStartOnly=true
LimitNOFILE=65535
Environment=CHAIN_ID='$CHAIN_ID'
Environment=GRPC_ENDPOINT='$GRPC_ENDPOINT'
Environment=WEBSOCKET_ENDPOINT='$WEBSOCKET_ENDPOINT'
Environment=EXCHANGE_SYMBOLS_MAP='$EXCHANGE_SYMBOLS_MAP'
Environment=FEEDER_MNEMONIC='$FEEDER_MNEMONIC'
Environment=VALIDATOR_ADDRESS='$VALIDATOR_ADDRESS'

[Install]
WantedBy=multi-user.target
EOF
```
## Delegate pricefeeder responsibility
### As a validator, if you'd like another account to post prices on your behalf (i.e. you don't want your validator mnemonic sending txs), you can delegate pricefeeder responsibilities to another nibi address.
```python
nibid tx oracle set-feeder $(nibid keys show pricefeeder-wallet -a) --from wallet
```
### Register and start the systemd service
```python
sudo systemctl daemon-reload && \
sudo systemctl enable pricefeeder && \
sudo systemctl start pricefeeder
```
### View pricefeeder logs
```python
journalctl -fu pricefeeder
```
### Successfull Log examples:
```python
Feb 27 19:09:33 pricefeeder[3632198]: {"level":"info","voting-period-height":405780,"tx-hash":"8FC418510A2BBFEF6E2FF17108B5D1C909B14DC0AFB22129A73C7373E649329F","time":1677521373,"message":"successfully forwarded prices"}
Feb 27 19:09:48 pricefeeder[3632198]: {"level":"info","voting-period":{"Height":405790},"time":1677521388,"message":"new voting period"}
Feb 27 19:09:48 pricefeeder[3632198]: {"level":"info","voting-period-height":405790,"vote":{"salt":"337","exchange_rates":"(unibi:unusd,0.066667000000000000)|(ubtc:unusd,23296.630000000000000000)|(ueth:unusd,1628.900000000000000000)|(uatom:unusd,12.760000000000000000)|(ubnb:unusd,301.740800000000000000)|(uusdc:unusd,0.999900000000000000)|(uusdt:unusd,1.000100000000000000)|(unibi:uusd,0.066667000000000000)|(ubtc:uusd,23314.000000000000000000)|(ueth:uusd,1629.900000000000000000)|(uatom:uusd,12.760000000000000000)|(ubnb:uusd,301.740800000000000000)|(uusdc:uusd,0.999900000000000000)|(uusdt:uusd,1.000100000000000000)","feeder":"nibi1w9d0u9ln9tx9dnn5qku9977jn2rtrupxxx0lrn","validator":"nibivaloper195w5wxp8hgqgz7schdukq7u5kadc2tnrsc00a0"},"time":1677521388,"message":"prepared vote message"}
Feb 27 19:09:50 pricefeeder[3632198]: {"level":"info","voting-period-height":405790,"tx-hash":"ACB94C8421AE056468BDAAE68655D86CEB87FB6881A526A31A7A2E98234C8D13","time":1677521390,"message":"successfully forwarded prices"}
Feb 27 19:10:05 pricefeeder[3632198]: {"level":"info","voting-period":{"Height":405800},"time":1677521405,"message":"new voting period"}
Feb 27 19:10:05 pricefeeder[3632198]: {"level":"info","voting-period-height":405800,"vote":{"salt":"3334","exchange_rates":"(unibi:unusd,0.066667000000000000)|(ubtc:unusd,23296.680000000000000000)|(ueth:unusd,1628.780000000000000000)|(uatom:unusd,12.760000000000000000)|(ubnb:unusd,301.740800000000000000)|(uusdc:unusd,0.999900000000000000)|(uusdt:unusd,1.000100000000000000)|(unibi:uusd,0.066667000000000000)|(ubtc:uusd,23296.680000000000000000)|(ueth:uusd,1628.780000000000000000)|(uatom:uusd,12.760000000000000000)|(ubnb:uusd,301.740800000000000000)|(uusdc:uusd,0.999900000000000000)|(uusdt:uusd,1.000100000000000000)","feeder":"nibi1w9d0u9ln9tx9dnn5qku9977jn2rtrupxxx0lrn","validator":"nibivaloper195w5wxp8hgqgz7schdukq7u5kadc2tnrsc00a0"},"time":1677521405,"message":"prepared vote message"}
Feb 27 19:10:06 pricefeeder[3632198]: {"level":"info","voting-period-height":405800,"tx-hash":"1E85AD6307DD8D734AAB6565CC57FB09F0EA0F7CFE0B8EE8AB50FC8A08B4111A","time":1677521406,"message":"successfully forwarded prices"}
Feb 27 19:10:22 pricefeeder[3632198]: {"level":"info","voting-period":{"Height":405810},"time":1677521422,"message":"new voting period"}
Feb 27 19:10:22 pricefeeder[3632198]: {"level":"info","voting-period-height":405810,"vote":{"salt":"9672","exchange_rates":"(unibi:unusd,0.066667000000000000)|(ubtc:unusd,23310.000000000000000000)|(ueth:unusd,1630.010000000000000000)|(uatom:unusd,12.760000000000000000)|(ubnb:unusd,301.737500000000000000)|(uusdc:unusd,0.999900000000000000)|(uusdt:unusd,1.000000000000000000)|(unibi:uusd,0.066667000000000000)|(ubtc:uusd,23307.000000000000000000)|(ueth:uusd,1629.200000000000000000)|(uatom:uusd,12.760000000000000000)|(ubnb:uusd,301.737500000000000000)|(uusdc:uusd,0.999900000000000000)|(uusdt:uusd,1.000000000000000000)","feeder":"nibi1w9d0u9ln9tx9dnn5qku9977jn2rtrupxxx0lrn","validator":"nibivaloper195w5wxp8hgqgz7schdukq7u5kadc2tnrsc00a0"},"time":1677521422,"message":"prepared vote message"}
Feb 27 19:10:23 pricefeeder[3632198]: {"level":"info","voting-period-height":405810,"tx-hash":"24B3B16FB8B6B047885928FB6A254D7B0B44E17E0DC6AF37B1F077F78C7099A9","time":1677521423,"message":"successfully forwarded prices"}
```
## Also you can check that your pricefeeder-wallet is doing transactions on chain at Chain Explorer
![image](https://user-images.githubusercontent.com/61777095/230730115-8b8f3278-acde-41f3-b4c4-f661c7c118fc.png)


