# Axelar Casablanca-1 Testnet Validator Node Setup

## Install Ubuntu 20.04 on a new server and login as root

## Install ``ufw`` firewall and configure the firewall

```
apt-get update
apt-get install ufw
ufw default deny incoming
ufw default allow outgoing
ufw allow 22
ufw allow 26656
ufw allow 50051
ufw enable
```

## Create a new User

```
# add user
adduser node

# add user to sudoers
usermod -aG sudo node

# login as user
su - node
```

## Install Prerequisites

```
sudo apt update
sudo apt install pkg-config build-essential libssl-dev curl jq git libleveldb-dev -y
sudo apt-get install manpages-dev -y

# install go
curl https://dl.google.com/go/go1.17.5.linux-amd64.tar.gz | sudo tar -C/usr/local -zxvf -

# Update environment variables to include go
cat <<'EOF' >>$HOME/.profile
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export GO111MODULE=on
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
EOF

source $HOME/.profile

# check go version
go version

# check gcc version
gcc --version
```

## Install Axelarate Community Github repo

```
git clone https://github.com/axelarnetwork/axelarate-community.git
cd axelarate-community
./scripts/setup-node.sh -n testnet-2
sudo cp -a /home/node/.axelar_testnet-2/bin/axelard /usr/bin
cd
```

## Download ``tofnd`` and move it to ``/usr/bin/`` folder.

```
wget https://github.com/axelarnetwork/tofnd/releases/download/v0.10.1/tofnd-linux-amd64-v0.10.1
mv tofnd-linux-amd64-v0.10.1 tofnd
chmod +x tofnd
sudo mv tofnd /usr/bin/
```

## Download latest Snapshot

Download the snapshot from https://bitszn.com/snapshots.html

```
wget https://snapshots.bitszn.com/snapshots/axelar/axelar.tar
```
## Move the downloaded Snapshot to ``.axelar_testnet-2`` folder and unzip the file inside ``data`` folder.
```
mkdir ~/.axelar_testnet-2/data
mv axelar.tar ~/.axelar_testnet-2/data/
cd ~/.axelar_testnet-2/data/
tar -xvf axelar.tar
```
## Start the node and let it Sync
```
axelard start Test_Node --home /home/node/.axelar_testnet-2
```
Let the blockchain sync. After the sync is completed, open a new terminal window and login to the server. Then, login as the user ```node``.

## Create the necessary wallets

```
axelard keys add broadcaster --home /home/node/.axelar_testnet-2
```
Give password for the above command and take a backup of the address and the Mnemonic keys and store it a safe location.

```
axelard keys add validator --home /home/node/.axelar_testnet-2
```
Give password for the above command and take a backup of the address and the Mnemonic keys and store it a safe location.

```
cd /home/node/.axelar_testnet-2
mkdir tofnd
cd
tofnd -m create -d /home/node/.axelar_testnet-2/tofnd
cd /home/node/.axelar_testnet-2/tofnd
cat export
```
Copy the Mnemonic keys and store it in a safe location.

Delete the ``export`` file.

```
rm export
cd
```

## Set Environment Variables

```
echo export CHAIN_ID=axelar-testnet-casablanca-1 >> $HOME/.profile
echo export MONIKER=PUT_YOUR_MONIKER_HERE >> $HOME/.profile
VALIDATOR_OPERATOR_ADDRESS=`axelard keys show validator --bech val --home /home/node/.axelar_testnet-2 --output json | jq -r .address`
BROADCASTER_ADDRESS=`axelard keys show broadcaster --home /home/node/.axelar_testnet-2 --output json | jq -r .address`
echo export VALIDATOR_OPERATOR_ADDRESS=$VALIDATOR_OPERATOR_ADDRESS >> $HOME/.profile
echo export BROADCASTER_ADDRESS=$BROADCASTER_ADDRESS >> $HOME/.profile
source $HOME/.profile
```

Stop the blockchain from the previous terminal window by pressing ``ctrl+c`` keys.


## Running the validator as a systemd unit

You have to create 3 services.

```
cd /etc/systemd/system
sudo nano axelard.service
```
Copy the following content into ``axelard.service`` and save it.

```
[Unit]
Description=Axelard Cosmos daemon
After=network-online.target

[Service]
User=<user>
ExecStart=/usr/bin/axelard start Test_Node --home /home/<user>/.axelar_testnet-2
Restart=on-failure
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
```

```
sudo nano vald.service
```

Copy the following content into ``vald.service`` and save it.

```
[Unit]
Description=Vald daemon
After=network-online.target
[Service]
User=<user>
ExecStart=/usr/bin/sh -c 'echo <password> | /usr/bin/axelard vald-start --validator-addr <validator-address> --log_level debug --chain-id axelar-testnet-casablanca-1 --home /home/<user>/.axelar_testnet-2'
Restart=on-failure
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
```

```
sudo nano tofnd.service
```

Copy the following content into ``tofnd.service`` and save it.


```
[Unit]
Description=Tofnd daemon
After=network-online.target

[Service]
User=<user>
ExecStart=/usr/bin/sh -c 'echo <password> | tofnd -m existing -d /home/<user>/.axelar_testnet-2/tofnd'
Restart=on-failure
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
```

Reload the daemon and enable to services.

```
sudo systemctl daemon-reload
sudo systemctl enable axelard
sudo systemctl enable vald
sudo systemctl enable tofnd
```

Start the services.

```
sudo systemctl start axelard
sudo systemctl start vald
sudo systemctl start tofnd
```

Checking the logs.

```
journalctl -u axelard.service -f -n 100
journalctl -u tofnd.service -f -n 100
journalctl -u vald.service -f -n 100
```

## Get ``axl`` tokens from the faucet

Get ``axl`` tokens from the casablanca faucet - https://faucet-casablanca.testnet.axelar.dev/

## Restart the services

Copy paste and execute the following commands 1 line at a time.

```
sudo kill -9 $(pgrep tofnd)
sudo kill -9 $(pgrep -f "axelard vald-start")
sudo systemctl restart axelard
sudo systemctl restart tofnd
sudo systemctl restart vald
```

## Register Broadcaster Proxy

```
axelard tx snapshot register-proxy $BROADCASTER_ADDRESS --from validator --home /home/node/.axelar_testnet-2 --gas auto --gas-prices 1.5uaxl --gas-adjustment 1.4 --chain-id axelar-testnet-casablanca-1
```

## Create and Register Your Validator Node

```
axelard tx staking create-validator --yes \
  --home /home/node/.axelar_testnet-2 \
  --amount 5000000uaxl \
  --moniker "Test_Node" \
  --website="https://www.xyz.com" \
  --details="Some Details" \
  --commission-rate="0.01" \
  --commission-max-rate="0.10" \
  --commission-max-change-rate="0.01" \
  --min-self-delegation="1" \
  --pubkey="$(axelard tendermint show-validator --home /home/node/.axelar_testnet-2)" \
  --from validator \
  -b block \
  --identity="D74433D32938F013" \
  --chain-id $CHAIN_ID \
  --gas auto \
  --gas-prices 1.5uaxl \
  --gas-adjustment 1.4
```  

## Register as EVM Chain Maintainer

Next step is to register as chain maintainer for EVM chains such as Ethereum. For now, Casablanca testnet supports Ethereum and Avalance. You can install and run your own nodes for Ethereum and Avalance or you can APIs from 3rd party services such as GetBlock, Infura, Quicknodes, Alchemy. We are using GetBlock as an example.

Open ``~/.axelar_testnet-2/config/config.toml`` file, scroll down to the end, edit the following section. You have to replace ``<API-KEY>`` with your respective API keys from GetBlock website.

```
[[axelar_bridge_evm]]
name = "Ethereum"
rpc_addr = "https://eth.getblock.io/ropsten/?api_key=<API-KEY>"
start-with-bridge = true

[[axelar_bridge_evm]]
name = "Avalanche"
rpc_addr = "https://avax.getblock.io/testnet/ext/bc/C/rpc?api_key=<API-KEY>"
start-with-bridge = true
```

Save the file and restart all the services.

```
cd
sudo kill -9 $(pgrep tofnd)
sudo kill -9 $(pgrep -f "axelard vald-start")
sudo systemctl restart axelard
sudo systemctl restart tofnd
sudo systemctl restart vald
```

Execute the following command to Register as EVM Chain Maintainer.

```
axelard tx nexus register-chain-maintainer avalanche ethereum --from broadcaster --home /home/node/.axelar_testnet-2 --gas auto --gas-prices 1.5uaxl --gas-adjustment 1.4 --chain-id axelar-testnet-casablanca-1
```

## Health Check

```
axelard health-check --tofnd-host localhost --operator-addr $VALIDATOR_OPERATOR_ADDRESS --home /home/node/.axelar_testnet-2
```

## Command to delegete more AXL tokens to the node

In the following example comamnd, we are delegating 1 AXL tokens to the validator node.

```
axelard tx staking delegate <validator-operator-address> 1000000uaxl --from validator --home /home/node/.axelar_testnet-2 --gas auto --gas-prices 1.5uaxl --gas-adjustment 1.4 --chain-id axelar-testnet-casablanca-1 -y
```

## Query balance of an address

```
axelard query bank balances <axelar-casablanca-address> --chain-id axelar-testnet-casablanca-1
```

## Backup Validator node file

Take a backup of the following files after you have created and registered your validator node successfully.

```
/home/node/.axelar_testnet-2/config/node_key.json
/home/node/.axelar_testnet-2/config/priv_validator_key.json
/home/node/.axelar_testnet-2/data/priv_validator_state.json
```





