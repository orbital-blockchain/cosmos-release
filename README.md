# Instructions for becoming an Orbital full node and/or validator

**Minimum hardware spec:**  
2CPU, 4GB RAM, 100GB storage. N.B. As the network grows, all three requirements will increase.

**The following instructions apply for Ubuntu 22.04 LTS 64-bit**  
Throughout, replace `<moniker>` with the name of your node.

### Once running, the blockchain can be interacted with using CLI commands

For full instructions on usage:

```console
./orbitald -h
```

## Full node

### 1. Update the local package list and install any available upgrades

```console
sudo apt-get update && sudo apt upgrade -y
```

### 2. Install toolchain and ensure accurate time synchronization

```console
sudo apt-get install make build-essential gcc git jq chrony -y
```

### 3. Add validator as system user

```console
sudo adduser <moniker>
```

```console
sudo usermod -aG sudo <moniker>
```

### 4. Log out and back in as the new user

```console
exit
```

### 5. Install Go

```console
wget https://go.dev/dl/go1.22.3.linux-amd64.tar.gz
```

```console
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.22.3.linux-amd64.tar.gz
```

### 6. Update profile

```console
pico ~/.profile
```

```console
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export GO111MODULE=on
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
export DAEMON_NAME=orbitald
export DAEMON_HOME=$HOME/.orbital
```

### 7. Apply the changes

```console
source ~/.profile
```

### 8. Check Go installation

```console
go version
```

```
go version go1.22.0 linux/amd64
```

### 9. Set up firewall

```console
sudo ufw default deny incoming &&
sudo ufw default allow outgoing &&
sudo ufw allow ssh &&
sudo ufw allow 26656/tcp &&
sudo ufw allow 26660/tcp
```

### 10. If the node would like to expose CometBFTs jsonRPC and Cosmos SDK GRPC and REST

Note: do not set these for full/validator node if using a sentry node. do set them for the sentry node if the sentry permits public access.

```console
sudo ufw allow 26657/tcp &&
sudo ufw allow 9090/tcp &&
sudo ufw allow 1317/tcp
```

### 11. Enable firewall

```console
sudo ufw enable
```

### 12. Download Orbital binary and config files

```console
git clone https://github.com/orbital-blockchain/cosmos-release
```

Move the relevant binary for your OS:

```console
sudo mv cosmos-release/binaries/orbital_linux_amd64.tar.gz .
```

Unzip:

```console
tar xzf orbital_linux_amd64.tar.gz
```

Binaries for other OS:

```
https://github.com/orbital-blockchain/cosmos-release/tree/main/binaries
```

### 13. Initialize the application and create app key

```console
./orbitald init <moniker> --chain-id orbital
```

```console
./orbitald keys add <moniker> --keyring-backend os
```

N.B. For local devnet, the following account mnemonic can be used since it will have tokens:

```console
./orbitald keys add <moniker> --keyring-backend os --recover
```

```
trick jump eight arrive machine oyster joy latin loyal supreme inner wire gospel obscure slot nature tornado rack peasant disease gravity critic woman car
```

Store keyring passphrase and mnemonic somewhere safe.

### 14a. Move genesis and config files

If running local devnet, remove existing config folder and replace:

```console
sudo rm -rf ~/.orbital/config
```

```console
sudo cp -r cosmos-release/devnet/config ~/.orbital/config
```

If not running local devnet, just copy relevant files.

Genesis file:

```console
sudo cp cosmos-release/config/genesis.json ~/.orbital/config/genesis.json
```

Lock genesis file:

```console
chmod a-wx ~/.orbital/config/genesis.json
```

Cosmos BFT config:

```console
sudo cp cosmos-release/config/config.toml ~/.orbital/config/config.toml
```

```console
sudo cp cosmos-release/config/app.toml ~/.orbital/config/app.toml
```

Remove repo:

```
rm -rf cosmos-release
```

### 14b. Update config files

Update moniker in config.toml:

```console
pico ~/.orbital/config/config.toml
```

```
moniker = "<moniker>"
```

For sentry, also update:

```
persistent_peers = "<node id>@<validator node private ip>:26656"
addr_book_strict = false
max_num_inbound_peers = 500
max_num_outbound_peers = 50
unconditional_peer_ids = "<validator node id>"
flush_throttle_timeout = "300ms"
private_peer_ids = "<validator node id>"
indexer = "null"
```

If sentry is being used, for validator node, also update:

```
persistent_peers = "<node id>@<node private ip>:26656"
addr_book_strict = false
unconditional_peer_ids = "<node id>"
private_peer_ids = "<node id>"
```

where <node id> can be obtained from ./orbitald tendermint show-node-id or ./orbitald status.

For full/sentry node:
Update app.toml for performance:

```console
pico ~/.orbital/config/app.toml
```

```
pruning = "custom"
pruning-keep-recent = "100"
pruning-interval = "10"
```

Update config.toml for performance:

```console
pico ~/.orbital/config/config.toml
```

```
max_num_inbound_peers = 500
max_num_outbound_peers = 50
flush_throttle_timeout = "300ms"
```

### 15. Back up keys

```console
~/.orbital/config/node_key.json
~/.orbital/config/priv_validator_key.json
```

### 16. Install Cosmovisor

```console
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@latest
```

```console
which cosmovisor
```

```
/home/<moniker>/go/bin/cosmovisor
```

### 17. Copy orbital binary to Cosmovisor folder

```console
mkdir -p $DAEMON_HOME/cosmovisor/genesis/bin && mkdir -p $DAEMON_HOME/cosmovisor/upgrades &&
cp $HOME/orbitald $DAEMON_HOME/cosmovisor/genesis/bin
```

### 18. Set up Cosmovisor service (remember to replace `<moniker>` throughout)

```console
sudo nano /etc/systemd/system/orbitald.service
```

```
[Unit]
Description=Orbital Daemon (cosmovisor)
After=network-online.target

[Service]
User=<moniker>
ExecStart=/home/<moniker>/go/bin/cosmovisor run start
Restart=always
RestartSec=3
LimitNOFILE=4096
Environment="DAEMON_NAME=orbitald"
Environment="DAEMON_HOME=/home/<moniker>/.orbital"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=false"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"

[Install]
WantedBy=multi-user.target
```

```console
sudo -S systemctl daemon-reload &&
sudo -S systemctl enable orbitald &&
sudo systemctl start orbitald &&
sudo systemctl status orbitald
```

```
     orbitald.service - Orbital Daemon (cosmovisor)
     Loaded: loaded (/etc/systemd/system/orbitald.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2024-04-01 08:51:22 UTC; 23ms ago
   Main PID: 12667 (cosmovisor)
      Tasks: 6 (limit: 4642)
     Memory: 5.7M
        CPU: 19ms
     CGroup: /system.slice/orbitald.service
             └─12667 /home/genesis/go/bin/cosmovisor run start

~$ systemd[1]: Started Orbital Daemon (cosmovisor).
```

### 19. Check status

```console
./orbitald status
```

### 20. Monitor logs CTRL +C to exit

```console
journalctl -fu orbitald
```

# Validator

**After ensuring that the node has caught up and has a balance of mepic:**

### 1. Get validator address and pubkey

Validator account address for receiving Orbitals:

```console
./orbitald keys show val
```

Validator pubkey for creating validator:

```console
./orbitald tendermint show-validator
```

### 2. Create validator JSON

N.B. Set all parameters accordingly, including pubkey from previous step, noting the minimum stake of 10,000,000 micro epic.

```console
echo '{
  "amount": "10000000mepic",
  "commission-max-change-rate": "0.1",
  "commission-max-rate": "0.20",
  "commission-rate": "0.1",
  "min-self-delegation": "1",
  "details": "",
  "pubkey": {"@type":"/cosmos.crypto.ed25519.PubKey","key":"94iUWgAqtERO6FbafafcIe5Pu2Oe0naoF85CdyTQzqc="},
  "moniker": "<moniker>"
}' > validator.json
```

### 3. Create validator

```console
./orbitald tx staking create-validator ~/validator.json \
  --chain-id orbital \
  --gas-prices 0mepic \
  --from <moniker>
```

### 4. Check status

```console
./orbitald status
```

```
block_height: "7028"
pagination:
  next_key: null
  total: "2"
validators:
- address: orbitalvalcons1ena4fn7f9ntq0jjzvknkm4kz9493amuvj70lyj
  proposer_priority: "91875"
  pub_key:
    '@type': /cosmos.crypto.ed25519.PubKey
    key: 5/bbFhKzKuBpca+l9YRF8CAB+9m6BC8qviCXVosOLfg=
  voting_power: "490000"
- address: orbitalvalcons1hcuav035262889ekzy4n2t6t5qt0grz0qpjccl
  proposer_priority: "-91875"
  pub_key:
    '@type': /cosmos.crypto.ed25519.PubKey
    key: 94iUWgAqtERO6FbafafcIe5Pu2Oe0naoF85CdyTQzqc=
  voting_power: "10"
```

### 5. Increase stake

By 10,000,000 mepic

```console
./orbitald keys show <moniker> --bech val
```

```console
./orbitald tx staking delegate <orbitalvaloper address> 10000000mepic --from <moniker> --chain-id orbital
```

# Unjailing validator

### 1. Get validator consensus address

```console
 ./orbitald keys show <moniker> --bech val
```

```
- address: orbitalvaloper1q5jydw8gsaagn60t3nxzvml3serggtflmv4hj9
  name: <moniker>
  pubkey: '{"@type":"/cosmos.crypto.secp256k1.PubKey","key":"Aj5xG5pjhR89jZgKTYcfWLUF/MdGI34jD7BUrNlib0z+"}'
  type: local
```

### 2. Check validator is not in validator set

```console
 ./orbitald query tendermint-validator-set
```

### 3. Check validator details

```console
./orbitald query staking validator orbitalvaloper1q5jydw8gsaagn60t3nxzvml3serggtflmv4hj9
```

```
validator:
  commission:
    commission_rates:
      max_change_rate: "100000000000000000"
      max_rate: "200000000000000000"
      rate: "100000000000000000"
    update_time: "2024-04-04T16:55:06.941818022Z"
  consensus_pubkey:
    type: tendermint/PubKeyEd25519
    value: +nrhFCCG5592/OKjw/1/AvBlnqCg7UelM9CuHWroXhI=
  delegator_shares: "111010101010101010101010101"
  description:
    moniker: val
  jailed: true
  min_self_delegation: "1"
  operator_address: orbitalvaloper1q5jydw8gsaagn60t3nxzvml3serggtflmv4hj9
  status: 2
  tokens: "109900000"
  unbonding_height: "6392"
  unbonding_ids:
  - "1"
  unbonding_time: "2024-04-25T17:03:48.759997998Z"
```

### 4. Check validator is up to date

```console
./orbitald status
```

```
"catching_up":false
"voting_power":"0"
```

### 6. Unjail validator

```console
./orbitald tx slashing unjail \
  --from=<moniker> \
  --chain-id=orbital \
  --yes
```

Check 2, 3 and 4 again to verify validator is unjailed.

# Installing a remote signer

It is recommended to separate the node keys from the node for extra security.

**Minimum hardware spec:**  
2CPU, 2GB RAM, 10GB storage.

### 1. Update server dependencies and install extras needed, and create new user

```console
sudo apt-get update && sudo apt upgrade -y && sudo apt install build-essential curl jq -y
```

```console
sudo adduser signer
```

```console
sudo usermod -aG sudo signer
```

Log out and back in as signer

### 2. Install Rust

```console
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

```console
source ~/.profile
```

### 3. Install Libusb

```console
sudo apt install libusb-1.0-0-dev
```

### 4. Setup

Cargo install:

```console
cargo install tmkms --features=softsign
tmkms init config
tmkms softsign keygen ./config/secrets/secret_connection_key
```

### 5. Migrate the validator key from the full node to the new tmkms instance

```console
scp <moniker>@<public ip of validator node>:~/.orbital/config/priv_validator_key.json ~/config/secrets
```

### 6. Import the validator key into tmkms

```console
tmkms softsign import ~/config/secrets/priv_validator_key.json ~/config/secrets/priv_validator_key
```

### 7. Safely store priv_validator_key.json offline and delete it from the validator node and the tmkms node.

```console
rm -rf priv_validator_key.json
```

### 8. Modifiy the tmkms.toml.

```console
pico ~/config/tmkms.toml
```

```
# CometBFT KMS configuration file

## Chain Configuration

### Cosmos Hub Network

[[chain]]
id = "orbital"
key_format = { type = "bech32", account_key_prefix = "orbitalpub", consensus_key_prefix = "orbitalvalconspub" }
state_file = "/home/signer/config/state/priv_validator_state.json"

## Signing Provider Configuration

### Software-based Signer Configuration

[[providers.softsign]]
chain_ids = ["orbital"]
key_type = "consensus"
path = "/home/signer/config/secrets/priv_validator_key"

## Validator Configuration

[[validator]]
chain_id = "orbital"
addr = "tcp://<public ip of validator>:26655"
secret_key = "/home/signer/config/secrets/secret_connection_key"
protocol_version = "v0.34"
reconnect = true
```

### 10. Create service and start KMS

```console
sudo nano /etc/systemd/system/tmkms.service
```

```
[Unit]
Description=tmkms
After=network-online.target
[Service]
User=signer
ExecStart=/home/signer/.cargo/bin/tmkms start -c /home/signer/config/tmkms.toml
Restart=on-failure
RestartSec=3
LimitNOFILE=4096
[Install]
WantedBy=multi-user.target
```

```console
sudo systemctl enable tmkms.service
```

```console
sudo systemctl start tmkms
```

### 11. Set the address of the tmkms instance in genesis node and disable local route to old key locations in config.toml

```console
pico ~/.orbital/config/config.toml
```

```
# priv_validator_key_file = "config/priv_validator_key.json"
# priv_validator_state_file = "data/priv_validator_state.json"
priv_validator_laddr = "tcp://0.0.0.0:26655"
```

### 12. Open port 26655 from tmkms

```console
sudo ufw allow from <public ip of signer> to any port 26655/tcp
```

### 12. Restart cosmovisor

```console
sudo systemctl restart orbitald &&
sudo systemctl status orbitald
```

### 13. Check status

```console
orbitald status
```

### 14. Monitor logs

```console
journalctl -fu orbitald
```

CTRL+C to exit
