
# INSTALL ENTRYPOINT NETWORK 



## Installation

update 
```bash
sudo apt update && apt upgrade -y
sudo apt install curl git jq lz4 build-essential unzip fail2ban ufw -y
```

Install Go 
```bash
ver="1.20.3"
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
go version
```

Buat Moniker
```bash
MONIKER="YOUR_MONIKER_NAME"
```

Custom Port
```bash
ENTRYPOINT_PORT=129
```

Custom Port
```bash
cd $HOME
wget https://snapshots.moonbridge.team/testnet/entrypoint/entrypointd
chmod +x entrypointd
mv entrypointd $HOME/go/bin/
entrypointd version --long | grep -e commit -e version
```

Konfigurasi
```bash
# Set node configuration
entrypointd config node tcp://localhost:${ENTRYPOINT_PORT}57
entrypointd config chain-id entrypoint-pubtest-2
entrypointd config keyring-backend test
entrypointd init $MONIKER --chain-id entrypoint-pubtest-2

# Download genesis and addrbook
curl -Ls https://snapshots.moonbridge.team/testnet/entrypoint/genesis.json > $HOME/.entrypoint/config/genesis.json
curl -Ls https://snapshots.moonbridge.team/testnet/entrypoint/addrbook.json > $HOME/.entrypoint/config/addrbook.json

# Set seeds and peers
SEEDS=""
PEERS=""
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.entrypoint/config/config.toml

# Setting minimum gas price
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.01ibc/8A138BC76D0FB2665F8937EC2BF01B9F6A714F6127221A0E155106A45E09BCC5\"|" $HOME/.entrypoint/config/app.toml

# Setting pruning
sed -i -e 's|^pruning *=.*|pruning = "custom"|' $HOME/.entrypoint/config/app.toml
sed -i -e 's|^pruning-keep-recent  *=.*|pruning-keep-recent = "100"|' $HOME/.entrypoint/config/app.toml
sed -i -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' $HOME/.entrypoint/config/app.toml
sed -i -e 's|^pruning-interval *=.*|pruning-interval = "10"|' $HOME/.entrypoint/config/app.toml

# Disable indexer
sed -i -e 's|^indexer *=.*|indexer = "null"|' $HOME/.entrypoint/config/config.toml

# Enable Prometheus
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.entrypoint/config/config.toml

# Setting custom ports
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:${ENTRYPOINT_PORT}58\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://0.0.0.0:${ENTRYPOINT_PORT}57\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:${ENTRYPOINT_PORT}60\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:${ENTRYPOINT_PORT}56\"%; s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${ENTRYPOINT_PORT}56\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":${ENTRYPOINT_PORT}66\"%" $HOME/.entrypoint/config/config.toml
sed -i -e "s%:1317%:${ENTRYPOINT_PORT}17%g; s%:8080%:${ENTRYPOINT_PORT}80%g; s%:9090%:${ENTRYPOINT_PORT}90%g; s%:9091%:${ENTRYPOINT_PORT}91%g; s%:8545%:${ENTRYPOINT_PORT}45%g; s%:8546%:${ENTRYPOINT_PORT}46%g; s%:6065%:${ENTRYPOINT_PORT}65%g" $HOME/.entrypoint/config/app.toml
```

Create service
```bash
sudo tee /etc/systemd/system/entrypointd.service > /dev/null <<EOF
[Unit]
Description=Entrypoint Node
After=network-online.target

[Service]
User=$USER
WorkingDirectory=$HOME/.entrypoint
ExecStart=$(which entrypointd) start --home $HOME/.entrypoint
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable entrypointd
```

jalankan node dan check logs
```bash
sudo systemctl start entrypointd && sudo journalctl -u entrypointd -f --no-hostname -o cat
```

Membuat wallet baru
```bash
entrypointd keys add wallet
```

jika sudah ada wallet bisa lewat recovery walet
```bash
entrypointd keys add wallet --recover
```

Check wallet balance 
```bash
entrypointd q bank balances $(entrypointd keys show wallet -a)
```

Membuat Validator
```bash
entrypointd tx staking create-validator \
  --amount 1000000uentry \
  --pubkey $(entrypointd tendermint show-validator) \
  --moniker "YOUR_MONIKER_NAME" \
  --identity "YOUR_KEYBASE_ID" \
  --details "YOUR_DETAILS" \
  --website "YOUR_WEBSITE_URL" \
  --security-contact "YOUR_EMAIL_ADDRESS" \
  --chain-id entrypoint-pubtest-2 \
  --commission-rate 0.10 \
  --commission-max-rate 0.20 \
  --commission-max-change-rate 0.01 \
  --min-self-delegation 1 \
  --from wallet \
  --gas-adjustment 1.4 \
  --gas auto \
  --gas-prices 0.01ibc/8A138BC76D0FB2665F8937EC2BF01B9F6A714F6127221A0E155106A45E09BCC5 \
  -y
```

# Sync

Sync
```bash
sudo systemctl stop entrypointd
cp $HOME/.entrypoint/data/priv_validator_state.json $HOME/.entrypoint/priv_validator_state.json.backup
entrypointd tendermint unsafe-reset-all --keep-addr-book --home $HOME/.entrypoint
```

Get and configure the state sync information
```bash
STATE_SYNC_RPC=https://entrypoint-test.rpc.moonbridge.team:443
LATEST_HEIGHT=$(curl -s $STATE_SYNC_RPC/block | jq -r .result.block.header.height)
SYNC_BLOCK_HEIGHT=$(($LATEST_HEIGHT - 1000))
SYNC_BLOCK_HASH=$(curl -s "$STATE_SYNC_RPC/block?height=$SYNC_BLOCK_HEIGHT" | jq -r .result.block_id.hash)

sed -i \
  -e "s|^enable *=.*|enable = true|" \
  -e "s|^rpc_servers *=.*|rpc_servers = \"$STATE_SYNC_RPC,$STATE_SYNC_RPC\"|" \
  -e "s|^trust_height *=.*|trust_height = $SYNC_BLOCK_HEIGHT|" \
  -e "s|^trust_hash *=.*|trust_hash = \"$SYNC_BLOCK_HASH\"|" \
  $HOME/.entrypoint/config/config.toml

mv $HOME/.entrypoint/priv_validator_state.json.backup $HOME/.entrypoint/data/priv_validator_state.json
```

Restart dan check logs
```bash
sudo systemctl start entrypointd && sudo journalctl -u entrypointd -f --no-hostname -o cat
```

