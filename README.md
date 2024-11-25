# K2 - The Koii Settlement Layer


<div align="center">
  
[![Docs](https://img.shields.io/badge/Docs-https%3A%2F%2Fdocs.koii.network-green)](https://docs.koii.network)
[![Latest version](https://img.shields.io/github/v/release/koii-network/k2-release?label=Latest&sort=semver)](https://github.com/koii-network/k2-release/releases)
[![Chat](https://img.shields.io/discord/776174409945579570?style=flat&label=Discord)](https://discord.gg/koii)
![Stars](https://img.shields.io/github/stars/koii-network/desktop-node?style=social)
![GitHub watchers](https://img.shields.io/github/watchers/koii-network/desktop-node?style=social)
[![Twitter](https://img.shields.io/twitter/follow/KoiiNetwork)](https://twitter.com/KoiiNetwork)
  
</div>

# Running a K2 Node

_At this time we only support Ubuntu 20.04 LTS. We offer macOS and Windows binaries however we do not have official guides on how to set up your validator environment on those operating systems._

## Staking

There is no minimum amount of KOII required to stake and participate in the voting process.

To participate in the voting process you must configure your system, start a validator, and configure your voting and stake accounts. This guide will show you how to do this.

## Quick Links

1. [System Requirements](README.md#system-requirements)
2. [System Setup](README.md#system-setup)
3. [Validator Setup](README.md#validator-setup)
4. [K2 Releases](https://github.com/koii-network/k2-release/releases)

## System Requirements
To run a K2 node, you need some KOII tokens and also possess the minimum memory (128 GB or 258 GB), computational (12 or 16 cores), and storage requirements.

There is no strict minimum amount of KOII tokens required to run the K2 node.

### Minimum Hardware Requirements

Here are the minimum hardware requirements for running a K2 node in terms of memory, compute, storage, and your operating system:

**1. Memory**

- 128GB, or more for consensus validator nodes
- 258GB, or more for RPC nodes

**2. Compute**

- 12 cores / 24 threads, or more @ minimum of 2.8GHz for consensus validator nodes
- 16 cores / 32 threads, or more for RPC nodes

**3. Storage**

For consensus validators:

- PCIe Gen3 x4 NVME SSD, or better
- Accounts: 500GB, or larger. High TBW (Total Bytes Written)
- Ledger: 1TB or larger. High TBW suggested

For RPC nodes:

- A larger ledger disk if longer transaction history is required, Accounts and ledger should not be stored on the same disk
- GPUs are not strictly necessary
- Network: 1 GBPS up and downlink speed, must be unshaped and unmetered

**4. Operating System**

Currently we are only supporting Ubuntu 20.04 for our validators. We do provide binaries for other operating systems however the operation of these binaries is not guaranteed.

## System Setup

<Description
  text="This section provides a guide for how to configure your Ubuntu system"
/>

Before continuing with this section you should ensure that your environment is up to date:

```bash
sudo apt update
sudo apt upgrade
```

After updating your environment you will need to install the required packages

```bash
sudo apt install libssl-dev libudev-dev pkg-config zlib1g-dev llvm clang
```

### Step 1: Create a new user

We recommend running the validator under a user that is not `root` for security reasons. Create a user to run the validator

```bash
sudo adduser koii
sudo usermod -aG sudo koii
```

Elevate into the user

```bash 
sudo su koii
cd ~
sudo chmod -R 775 /home/koii
```

### Step 2: Install the Koii software (Now version v1.16.4)

We host an install script that will install and configure the Koii validator software. Run it with the following command

```bash
sh -c "$(curl -sSfL https://raw.githubusercontent.com/koii-network/k2-release/master/k2-install-init_v1.16.4.sh)"
echo 'export PATH="/home/koii/.local/share/koii/install/active_release/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
koii config set --url https://testnet.koii.network
koii config get

```

This scipt will install and configure the validator software with an identity key and the `koii` cli configured for `testnet`. It is important to note that this identity key created IS NOT your validator identity. If you have a private key which is funded for staking with a validator you can replace the one generated with this script. 

If everything is configured correctly you can test it by running `koii balance` which will return the balance of the local key.

### Step 3: Open port and run the System Tuner

Open port
```bash
sudo ufw allow ssh 
sudo ufw allow 10000:10500/udp
sudo ufw allow 10000:10500/tcp
sudo ufw allow 10899/tcp
sudo ufw allow 10900/tcp
```
Check Port and ping

```bash
sudo ufw status
curl -I https://testnet.koii.network
sudo netstat -tuln | grep LISTEN
```
This will configure certain aspects of your system to better support the validator.

```bash
koii-sys-tuner --user koii
```

## Validator Setup

The following guide describes how to setup a validator on Ubuntu.

### Identity Setup
You will need to create the following keys on your system:

```bash
koii-keygen new --outfile ~/validator-keypair.json
koii-keygen new --outfile ~/withdrawer-keypair.json
koii-keygen new --outfile ~/stake-account-keypair.json
koii-keygen new --outfile ~/vote-account-keypair.json

```
#### Keypairs

## validator-keypair.json : Identity of the validator on the network. Copy this to the remote validator server at /home/koii/validator-keypair.json

## vote-account-keypair.json : Voting account on the network. Copy this to the remote validator server at /home/koii/vote-account-keypair.json

## stake-account-keypair.json : Keypair for your staking wallet. Store in a secure location not on the validator since this will hold the wallet that you delegate stake from

## authorized-withdrawer-keypair.json : Authorized withdrawer keypair, allowed to withdraw funds from your validator vote account. Store in a secure location not on the validator since this controls your vote account

Danger: The authorized withdrawer keypair is the ultimate authority over your validator. This keypair will be able to withdraw from your vote account and will have additional permission to change all other aspects of your vote account. Anyone in possession of it can permanently take control of your vote account and make any changes as they please.

### Step 1: Create a Systemctl Service File for the Validator

Write a service configuration using the editor of your choice (nano, vim, etc). Do this as a system user with root permissions, not your validator user.

```bash
sudo nano /etc/systemd/system/systuner.service
```
Paste the service configuration below into your editor.

```bash
[Unit]
Description=Koii System Tuner
After=network.target
[Service]
Type=simple
Restart=on-failure
RestartSec=1
ExecStart=/home/koii/.local/share/koii/install/active_release/bin/koii-sys-tuner --user koii
[Install]
WantedBy=multi-user.target
```

Save and close your editor.

**Start and Enable the service**

```bash
sudo systemctl start systuner
sudo systemctl enable systuner

```

```bash
sudo nano /home/koii/validator.sh

```

Paste the service configuration below into your editor.

```makefile
#!/bin/sh
exec /home/koii/.local/share/koii/install/active_release/bin/koii-validator \
--identity /home/koii/validator-keypair.json  \
--vote-account /home/koii/vote-account-keypair.json \
--ledger /home/koii/ledger/ledgerdb \
--accounts /home/koii/accounts/accountdb \
--log /home/koii/koii-rpc.log \
--rpc-bind-address 0.0.0.0 \
--rpc-port 10899 \
--gossip-port 10001 \
--dynamic-port-range 10002-10500 \
--enable-rpc-transaction-history \
--enable-cpi-and-log-storage \
--known-validator Bs3LDTq3rApDqjU4vCfDrQFjixHk1cW8rYoyc47bCog6 \
--entrypoint entrypoint-1.testnet.koii.network:10001 \
--entrypoint entrypoint-2.testnet.koii.network:10001 \
--rpc-faucet-address rpc-faucet.testnet.koii.network:9900 \
--init-complete-file /home/koii/init-completed \
--no-wait-for-vote-to-start-leader \
--enable-extended-tx-metadata-storage \
--maximum-full-snapshots-to-retain 20 \
--maximum-incremental-snapshots-to-retain 20 \
--limit-ledger-size 200000000 \
--only-known-rpc \
--wal-recovery-mode skip_any_corrupted_record
--expected-genesis-hash 3J1UybSMw4hCdTnQoVqVC3TSeZ4cd9SkrDQp3Q9j49VF
--expected-bank-hash 2Yvcz1QWRemddmoFhumBESUzeZiepXA8DZu3g2Z9Kh2J
--expected-shred-version 9890
```
Save and close your editor.

```bash
sudo nano /etc/systemd/system/koii-validator.service
```

Paste the service configuration below into your editor.

```bash
[Unit]
Description=Koii Validator
After=network.target
Wants=systuner.service
StartLimitIntervalSec=0
[Service]
User=koii
Group=koii
LimitNOFILE=1000000
LogRateLimitIntervalSec=0
Environment="PATH=/home/koii/.local/share/koii/install/active_release/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games"
ExecStart=/home/koii/validator.sh
Restart=on-failure
RestartSec=10
[Install]
WantedBy=multi-user.target
```

Save and close your editor.

```bash
koii config set --url https://testnet.koii.network --keypair ~/validator-keypair.json

```

### Step 2: Enable and Start the Koii Validator Service

Enable the service

```bash
sudo systemctl enable koii-validator.service
```

Start the service

```bash
sudo systemctl start koii-validator.service
```

Check the service status

```bash
sudo systemctl status koii-validator.service
```
(I think! Your node must be synced all blockchain at block height and then created next step)

### Step 3: Create a Vote Account

_**You will need your validator keypair to be funded with KOII tokens and have the validator service running before continuing.**_

For the remainder of the steps please elevate your user to your validator account.

```bash
sudo su koii
```

Using the keys created in the first portion of this guide, create a vote account. (transfer 2 koii to wallet validator-keypair.json)

```bash
koii create-vote-account ~/vote-account-keypair.json ~/validator-keypair.json ~/withdrawer-keypair.json
```

### Step 4: Create a Stake Account (Transfer amount Koii from other wallet that do you want staking make validator Active)

Create the staking account using the validator's identity keypair and the authorized withdrawer keypair:

```bash
koii create-stake-account ~/stake-account-keypair.json <AMOUNT_TO_STAKE> --stake-authority ~/validator-keypair.json --withdraw-authority ~/authorized-withdrawer-keypair.json
```

Where `<AMOUNT_TO_STAKE>` is the number of tokens you want to stake with.

Delegate

```bash
koii delegate-stake ~/stake-account-keypair.json ~/vote-account-keypair.json --stake-authority ~/validator-keypair.json --force

```
Check status delegate

```bash
koii stake-account ~/stake-account-keypair.json

```

### Step 5: Play Catchup

Make sure your validator is caught up with the network.

```bash
koii catchup ~/validator-keypair.json
```

### Step 6: More staking
Transfer some koii do you want staking to Wallet validator-keypair.json and then creating a new account stake

```bash
koii-keygen new --outfile <name>.json
```
```bash
koii create-stake-account <name>.json <amount> --stake-authority validator-keypair.json --withdraw-authority authorized-withdrawer-keypair.json
```
```bash
koii delegate-stake <name>.json vote-account-keypair.json --stake-authority validator-keypair.json
```
```bash
koii stake-account <name>.json
```

### Some CLI useful

```bash
sudo journalctl -u koii-validator.service -f
```

```bash
koii-watchtower
```

```bash
tail -f koii-rpc.log
```

```bash
tail -f -n2000 ~/koii-rpc.log
```
Or

```bash
tail -f -n2000 ~/koii-rpc.log | grep "WARN\|ERROR"
```

Wallet:

```bash
koii balance <path_to_wallet>
```
Deactive (stop staking)

```bash
koii deactivate-stake --stake-authority validator-keypair.json <your-stake-wallet.json> --fee-payer validator-keypair.json

```
## After deactive, i must be wait end epoch present (about 12 h/ 24h) and then next step

Withdraw

```bash
koii withdraw-stake --withdraw-authority authorized-withdrawer-keypair.json <your-stake-wallet.json> <public-address-you-want to receiv koii> <amount> --fee-payer validator-keypair.json

```
Check Reward when you run validator (validator always synced status): choise tab reward, after 12h of epoch you will have reward

```bash
https://explorer.koii.live/address/<vote-account-keypair.json>

```





