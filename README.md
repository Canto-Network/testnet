## Canto 7701 Testnet
### Overview
Genesis file:
- `wget https://raw.githubusercontent.com/Canto-Network/testnet/main/genesis.json`

Public RPC:
-  `canto-testnet.plexnode.wtf`

### Node Setup

This guide is based on the [Quickstart Guide](https://docs.canto.io/canto-node/validators/quickstart-guide) for Canto mainnet validator nodes. It is adapted to use the testnet genesis file from this repo and the correct testnet chain ID (7701).

Note that this testnet was launched with the [v5.0.0 Canto binary](https://github.com/Canto-Network/Canto/tree/v5.0.0). You must start with this binary if syncing from genesis; otherwise, use the latest binary.

#### 1. Install Dependencies

Install dependencies (Ubuntu):

```sh
sudo snap install go --classic
sudo apt-get install git
sudo apt-get install gcc
sudo apt-get install make
```

#### 2. Install `cantod`

Clone the official repo and install the v5.0.0 binary:

```sh
git clone https://github.com/Canto-Network/Canto.git
cd Canto
git checkout v5.0.0
make install
sudo mv $HOME/go/bin/cantod /usr/bin/
```

Generate and store keys:

```sh
cantod keys add <key_name>
```

To recover keys from an existing mnemonic, use the `--recover` flag.

#### 3. Initialize Validator

Initialize the node and download the genesis file:

```sh
cantod init <MONIKER> --chain-id canto_7701-1
cd ~/.cantod/config
rm genesis.json
wget https://raw.githubusercontent.com/Canto-Network/testnet/main/genesis.json
```

Replace `<moniker>` with whatever you'd like to name your validator.

#### 4. Edit Config

```sh
# Add seed peer to config.toml
sed -i 's/seeds = ""/seeds = "TBD"/g' $HOME/.cantod/config/config.toml

# Set minimum gas price in app.toml
sed -i 's/minimum-gas-prices = "0acanto"/minimum-gas-prices = "0.0001acanto"/g' $HOME/.cantod/config/app.toml
```

#### 5. Create systemd Service

Create the systemd service file:

```sh
sudo nano /etc/systemd/system/cantod.service
```

Copy and paste the following configuration and save:

```sh
[Unit]
Description=Canto Node
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/
ExecStart=/usr/bin/cantod start --trace --log_level info --json-rpc.api eth,txpool,personal,net,debug,web3 --api.enable
Restart=on-failure
StartLimitInterval=0
RestartSec=3
LimitNOFILE=65535
LimitMEMLOCK=209715200

[Install]
WantedBy=multi-user.target
```

#### 6. Start Node

```sh
# Reload service files
sudo systemctl daemon-reload

# Create the symlink
sudo systemctl enable cantod.service

# Start the node
sudo systemctl start cantod

# Show logs
journalctl -u cantod -f
```

You should then get several lines of log files, which may include an `INVALIDARGUMENT` error causing the service to exit. This is expected; `Ctrl + C` out and follow the next steps.

#### 7. Create Validator Transaction

Modify the following items below, removing the `<>`

* `<KEY_NAME>` should be the same as `<key_name>` when you followed the steps above in creating or restoring your key.
* `<VALIDATOR_NAME>` is whatever you'd like to name your node
* `<DESCRIPTION>` is whatever you'd like in the description field for your node
* `<SECURITY_CONTACT_EMAIL>` is the email you want to use in the event of a security incident
* `<YOUR_WEBSITE>` the website you want associated with your node
* `<TOKEN_DELEGATION>` is the amount of tokens staked by your node (minimum `1acanto`)

```sh
cantod tx staking create-validator \
--from <KEY_NAME> \
--chain-id canto_7701-1 \
--moniker="<VALIDATOR_NAME>" \
--commission-max-change-rate=0.01 \
--commission-max-rate=1.0 \
--commission-rate=0.05 \
--details="<DESCRIPTION>" \
--security-contact="<SECURITY_CONTACT_EMAIL>" \
--website="<YOUR_WEBSITE>" \
--pubkey $(cantod tendermint show-validator) \
--min-self-delegation="1" \
--amount <TOKEN_DELEGATION>acanto \
--fees 30000000000000000acanto \
--gas 300000
```

Your validator wallet must contain a non-zero amount of native testnet $CANTO in order to send the validator transaction. To get some, follow these steps:

1. Run `cantod debug addr $(cantod keys show <key_name> -a)` to see your validator's Bech32 and 0x addresses.
2. Request testnet Canto to this address from the #canto-testnet-faucet channel in the [Canto Discord](https://discord.gg/canto).
3. Alternatively, ask a validator who already has native $CANTO to send funds to the Bech32 Acc address.

#### 8. Update Binary

State breaking software upgrades took place at blocks:

* Blockheight TBD (v6.0.0)

Upon reaching these blocks while syncing, the node will halt and throw an error every time it restarts until you update the binary. To do so, follow these steps:

```shell
# Stop cantod
sudo systemctl stop cantod

# Delete old binary from path and install new binary (run in /Canto/ folder)
git checkout v6.0.0
sudo rm /usr/bin/cantod
make install
sudo mv $HOME/go/bin/cantod /usr/bin/

# Restart
sudo systemctl start cantod
```

For future binary upgrades, you will need to `git pull` to fetch the updated binary before you attempt to install it.