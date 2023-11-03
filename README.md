Not: Need ubuntu 22
### UPDATE SYSTEM AND INSTALL BUILD TOOLS
```
sudo apt -q update
sudo apt -qy install curl git jq lz4 build-essential
sudo apt -qy upgrade
```
### INSTALL GO
```
sudo rm -rf /usr/local/go
curl -Ls https://go.dev/dl/go1.20.10.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
eval $(echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/golang.sh)
eval $(echo 'export PATH=$PATH:$HOME/go/bin' | tee -a $HOME/.profile)
```
### Download project binaries
```
mkdir -p /root/.routerd/cosmovisor/genesis/bin
git clone https://github.com/router-protocol/router-chain-releases
cd router-chain-releases/linux
tar -xvf routerd.tar
mkdir -p /root/.routerd/cosmovisor/upgrades/v1.2.1-to-v1.2.2/bin
cp /root/router-chain-releases/linux/routerd /root/.routerd/cosmovisor/upgrades/v1.2.1-to-v1.2.2/bin/
mv /root/router-chain-releases/linux/routerd /root/.routerd/cosmovisor/genesis/bin/routerd
chmod +x /root/.routerd/cosmovisor/genesis/bin/routerd
```
### Create application symlinks
```
sudo ln -s /root/.routerd/cosmovisor/genesis /root/.routerd/cosmovisor/current -f
sudo ln -s /root/.routerd/cosmovisor/current/bin/routerd /usr/bin/routerd -f
```

### Download and install Cosmovisor
```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0
```
### Create service
```
sudo tee /etc/systemd/system/routerd.service > /dev/null << EOF
[Unit]
Description=router node service
After=network-online.target

[Service]
User=root
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.routerd"
Environment="DAEMON_NAME=routerd"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/root/.routerd/cosmovisor/current/bin"

[Install]
WantedBy=multi-user.target
EOF

```
```
sudo systemctl daemon-reload
sudo systemctl enable routerd.service
```
```
sudo wget -P /usr/lib https://github.com/CosmWasm/wasmvm/raw/main/internal/api/libwasmvm.x86_64.so
```
### Set node configuration
```
routerd config chain-id router_9601-1
routerd config keyring-backend file
routerd config node tcp://localhost:15257
```
### Initialize the node
```
routerd init corenode --chain-id router_9601-1
```
### Download genesis and addrbook
```
curl -Ls https://ss-t.router.nodestake.top/genesis.json > $HOME/.routerd/config/genesis.json 
curl -Ls https://ss-t.router.nodestake.top/addrbook.json > $HOME/.routerd/config/addrbook.json 
```
### Add seeds
```
sed -i -e "s|^seeds *=.*|seeds = \"89ec0f07f0ccb61ec19fb8256043cf92e73abd2b@15.206.157.168:26656,50dc3cca9f3b3f969b812e5760bcaf652aaecc01@43.205.136.8:26656,3df6cb2db301288c492f9ace1b88360e0504b15a@13.235.115.79:26656\"|" $HOME/.routerd/config/config.toml

sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0router\"|" $HOME/.routerd/config/app.toml
```

### Set pruning
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  /root/.routerd/config/app.toml
```
### Set custom ports
```
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:15258\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:15257\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:15260\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:15256\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":15266\"%" /root/.routerd/config/config.toml
sed -i -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:15217\"%; s%^address = \":8080\"%address = \":15280\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:15290\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:15291\"%; s%:8545%:15245%; s%:8546%:15246%; s%:6065%:15265%" /root/.routerd/config/app.toml
```
### Start
```
sudo systemctl start routerd.service && sudo journalctl -u routerd.service -f --no-hostname -o cat
```

### Snap
```
sudo systemctl stop routerd

cp $HOME/.routerd/data/priv_validator_state.json $HOME/.routerd/priv_validator_state.json.backup

rm -rf $HOME/.routerd/data

rm -rf $HOME/.routerd/wasm

SNAP_NAME=$(curl -s https://ss-t.router.nodestake.top/ | egrep -o ">20.*\.tar.lz4" | tr -d ">")

curl -o - -L https://ss-t.router.nodestake.top/${SNAP_NAME}  | lz4 -c -d - | tar -x -C $HOME/.routerd

mv $HOME/.routerd/priv_validator_state.json.backup $HOME/.routerd/data/priv_validator_state.json


sudo systemctl restart routerd
journalctl -u routerd -fo cat
```
--------------------------------------
#### Upgrade
```
cd $HOME
rm -rf router-chain-releases
git clone https://github.com/router-protocol/router-chain-releases
cd router-chain-releases/linux
tar -xvf routerd.tar
```
# Prepare binaries for Cosmovisor
```
mkdir -p /root/.routerd/cosmovisor/upgrades/v1.2.1-to-v1.2.2/bin
mv /root/router-chain-releases/linux/routerd /root/.routerd/cosmovisor/upgrades/v1.2.1-to-v1.2.2/bin/
```
