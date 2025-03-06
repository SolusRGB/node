# Running a node

## Machine Specs
Recommended minimum hardware: 4 CPU cores, 32 GB RAM, 200 GB disk.

Currently only Ubuntu 24.04 is supported.

Ports 4001 and 4002 are used for gossip and must be open to the public. Otherwise the node IP address will be deprioritized by peers in the p2p network.

For lowest latency, run the node in Tokyo, Japan.

---

## Setup

### Configure Chain

For **testnet**:
```bash
echo '{"chain": "Testnet"}' > ~/visor.json
```

For **mainnet**:
```bash
echo '{"chain": "Mainnet"}' > ~/visor.json
```

### Download the Visor Binary

For **testnet**:
```bash
curl https://binaries.hyperliquid-testnet.xyz/Testnet/hl-visor > ~/hl-visor && chmod a+x ~/hl-visor
```

For **mainnet**:
```bash
curl https://binaries.hyperliquid-mainnet.xyz/Mainnet/hl-visor > ~/hl-visor && chmod a+x ~/hl-visor
```

---

## Verify Signed Binaries

Binaries are signed for extra security. The public key is found at `pub_key.asc` in this repo. Import this key:
```bash
gpg --import pub_key.asc
```

Verify any (signature, binary) pair manually. Signatures are located at `{binary}.asc`.

For example, download and verify the testnet binary:
```bash
curl https://binaries.hyperliquid-testnet.xyz/Testnet/hl-visor.asc > hl-visor.asc
gpg --verify hl-visor.asc hl-visor
```

For mainnet, use the corresponding URL:
```bash
curl https://binaries.hyperliquid-mainnet.xyz/Mainnet/hl-visor.asc > hl-visor.asc
gpg --verify hl-visor.asc hl-visor
```

> **Note:** `hl-visor` will automatically verify `hl-node` and will not upgrade on verification failure. The public key must be imported (or signed with `gpg --sign-key`) to avoid warnings.

---

## Running Non-Validator

Run the visor with:
```bash
~/hl-visor run-non-validator
```
It may take a while as the node navigates the network to find an appropriate peer to stream from. Logs like `applied block X` mean the node should be streaming live data.

> **Tip:** Ensure that your chain configuration (in `~/visor.json`) is set appropriately for testnet or mainnet.

---

## Reading L1 Data

The node process writes data to `~/hl/data`. With default settings, the network generates around 20 GB of logs per day; therefore, it is recommended to archive or delete old files.

- **Transaction Blocks:**  
  Blocks parsed as transactions are streamed to:
  ```
  ~/hl/data/replica_cmds/{start_time}/{date}/{height}
  ```

- **State Snapshots:**  
  State snapshots are saved every 10,000 blocks to:
  ```
  ~/hl/data/periodic_abci_states/{date}/{height}.rmp
  ```
  
  The state can be translated to JSON for examination:
  ```bash
  ./hl-node --chain Testnet translate-abci-state ~/hl/data/periodic_abci_states/{date}/{height}.rmp /tmp/out.json
  ```
  For mainnet, substitute `Testnet` with `Mainnet`:
  ```bash
  ./hl-node --chain Mainnet translate-abci-state ~/hl/data/periodic_abci_states/{date}/{height}.rmp /tmp/out.json
  ```

---

## Flags

When running nodes (validator or non-validator), you can enable several flags:

- `--write-trades`: Streams trades to `~/hl/data/node_trades/hourly/{date}/{hour}`.
- `--write-order-statuses`: Writes every L1 order status to `~/hl/data/node_order_statuses/hourly/{date}/{hour}`.
- `--replica-cmds-style`: Configures what is written down to `~/hl/data/replica_cmds/{start_time}/{date}/{height}`. Possible values:
  - `actions` (default)
  - `actions-and-responses`
  - `recent-actions` (preserves only the two latest height files)
- `--serve-eth-rpc`: Enables the EVM RPC. See the following section.

For example, to run a non-validator with all flags enabled:
```bash
~/hl-visor run-non-validator --write-trades --write-order-statuses --serve-eth-rpc
```

> **Note:** These flags work regardless of the chain; simply ensure your configuration (`~/visor.json`) and command-line flags use the correct chain (Testnet or Mainnet).

---

## EVM

Enable the EVM RPC by passing the `--serve-eth-rpc` flag:
```bash
~/hl-visor run-non-validator --serve-eth-rpc
```

Once running, you can send requests (the same for both testnet and mainnet):
```bash
curl -X POST --header 'Content-Type: application/json' --data '{"jsonrpc":"2.0","method":"eth_getBlockByNumber","params":["latest",false],"id":1}' http://localhost:3001/evm
```

---

## Delegation

For **testnet**, the native token is **HYPE** with token address:
```
0x7317beb7cceed72ef0b346074cc8e7ab
```

For **mainnet**, replace the token address with the appropriate mainnet address (if different) and use the same delegation commands.

Delegations occur from the staking balance (separate from the spot balance). To move tokens into the staking balance:
```bash
./hl-node --chain Testnet --key <delegator-wallet-key> staking-deposit <wei>
```
For mainnet, replace `Testnet` with `Mainnet`:
```bash
./hl-node --chain Mainnet --key <delegator-wallet-key> staking-deposit <wei>
```

Delegate tokens by running:
```bash
./hl-node --chain Testnet --key <delegator-wallet-key> delegate <validator-address> <amount-in-wei>
```
For mainnet:
```bash
./hl-node --chain Mainnet --key <delegator-wallet-key> delegate <validator-address> <amount-in-wei>
```

To undelegate, add the `--undelegate` flag:
```bash
./hl-node --chain Testnet --key <delegator-wallet-key> delegate <validator-address> <amount-in-wei> --undelegate
```
For mainnet, again use `--chain Mainnet`.

To view delegations:
```bash
curl -X POST --header "Content-Type: application/json" --data '{ "type": "delegations", "user": <delegator-address>}' https://api.hyperliquid-testnet.xyz/info
```
For mainnet, use the corresponding API endpoint (if provided).

Staking withdrawals (subject to a 5-minute unbonding period) are initiated with:
```bash
./hl-node --chain Testnet --key <delegator-wallet-key> staking-withdrawal <wei>
```
And similarly for mainnet:
```bash
./hl-node --chain Mainnet --key <delegator-wallet-key> staking-withdrawal <wei>
```

---

## Running a Validating Node

The non-validating node setup above is a prerequisite for running a validating node.

### Generate Config

Generate two wallets: a **validator wallet** and a **signer wallet** (use cryptographically secure keys, e.g. via `openssl rand -hex 32`). The validator wallet is "cold" (stores funds and receives delegation rewards) while the signer wallet is "hot" (used only for signing consensus messages). For simplicity, these can be the same wallet.

Configure the signer wallet:
```bash
echo '{"key": "<signer-key>"}' > ~/hl/hyperliquid_data/node_config.json
```
Keep both `<signer-key>` and `<validator-key>` safe.

### Ensure Validator User Exists

Both the signer address and the validator address must have a non-zero perps USDC balance to participate in consensus. Print the addresses using:
```bash
~/hl-node --chain Testnet --key <signer-key> print-address
~/hl-node --chain Testnet --key <validator-key> print-address
```
For mainnet, replace `Testnet` with `Mainnet` in these commands.

### Join Network

For **testnet**, the validator set is entirely permissionless.

Register your public IP and signer address along with your display name and description. On testnet, you must self-delegate 10,000 (i.e. 1000000000000 wei) to run the validator:
```bash
~/hl-node --chain Testnet --key <validator-key> send-signed-action '{"type": "CValidatorAction", "register": {"profile": {"node_ip": {"Ip": "1.2.3.4"}, "signer": "<signer-address>", "name": "...", "description": "..." }, "initial_wei": 1000000000000}}'
```

For **mainnet**, use:
```bash
~/hl-node --chain Mainnet --key <validator-key> send-signed-action '{"type": "CValidatorAction", "register": {"profile": {"node_ip": {"Ip": "1.2.3.4"}, "signer": "<signer-address>", "name": "...", "description": "..." }, "initial_wei": 1000000000000}}'
```

Make sure ports 4000-4010 are open to other validators (currently only ports 4001-4006 are used, though additional ports in the range may be used in the future). Either open the ports publicly or configure your firewall to allow traffic from validators (found in `c_staking` in the state snapshots).

### Run the Validator

For **testnet**, run the validator using the visor binary to pick up updates:
```bash
curl https://binaries.hyperliquid-testnet.xyz/Testnet/hl-visor > hl-visor && ./hl-visor run-validator
```

For **mainnet**, run:
```bash
curl https://binaries.hyperliquid-mainnet.xyz/Mainnet/hl-visor > hl-visor && ./hl-visor run-validator
```

> **Debugging Tip:** To troubleshoot, it is sometimes easier to run:
> ```bash
> ./hl-node --chain Testnet run-validator
> ```
> or for mainnet:
> ```bash
> ./hl-node --chain Mainnet run-validator
> ```
> This command shows stderr directly and disables automatic restarts.

When bootstrapping, the validator starts with a non-validator process. To speed this up, you can specify a known reliable peer:
```bash
echo '{ "root_node_ips": [{"Ip": "1.2.3.4"}], "try_new_peers": false, "chain": "Testnet" }' > ~/override_gossip_config.json
```
For mainnet, change the chain to `"Mainnet"`:
```bash
echo '{ "root_node_ips": [{"Ip": "1.2.3.4"}], "try_new_peers": false, "chain": "Mainnet" }' > ~/override_gossip_config.json
```

### Begin Validating

Initially, registering or changing the IP automatically jails the validator. When you see the expected outputs streaming to `~/hl/data/node_logs/consensus/hourly/{date}/{hour}`, send the following action to begin participating in consensus:

For **testnet**:
```bash
~/hl-node --chain Testnet --key <signer-key> send-signed-action '{"type": "CSignerAction", "unjailSelf": null}'
```

For **mainnet**:
```bash
~/hl-node --chain Mainnet --key <signer-key> send-signed-action '{"type": "CSignerAction", "unjailSelf": null}'
```

To exit consensus, "self jail" by running:

For **testnet**:
```bash
~/hl-node --chain Testnet --key <signer-key> send-signed-action '{"type": "CSignerAction", "jailSelf": null}'
```

For **mainnet**:
```bash
~/hl-node --chain Mainnet --key <signer-key> send-signed-action '{"type": "CSignerAction", "jailSelf": null}'
```

### Jailing

Validators that fall behind in performance or connectivity are automatically jailed. Once jailed, a validator can only be unjailed through the `unjailSelf` action (once the L1 time passes the "jailed until" time). Self-jailing does not extend the duration of jailing.

For debugging jailing issues, check stdout and logs in `~/hl/data/node_logs/status/` for connectivity or latency problems.

### Alerting

Validators are encouraged to set up alerting to maintain optimal uptime. For example, to configure Slack alerts on testnet:
```bash
echo '{"testnet_slack_channel": "C000...", "slack_key": "Bearer xoxb-..."}' > ~/hl/api_secrets.json
```
And test it with:
```bash
~/hl-node --chain Testnet send-slack-alert "hello hyperliquid"
```

For **mainnet**, you might use a similar configuration, possibly with a key or channel specific to mainnet.

---

## Logs

The directory `node_logs/consensus` contains messages sent and received by the consensus algorithm—useful for debugging. For example, to check whether Vote messages were sent to validator `0x5ac9...` around `2024-12-10T09:25`:
```bash
grep destination...0x5ac9 ~/hl/data/node_logs/consensus/hourly/20241210/9 | grep T09:25 | grep Vote
```

If a validator experiences timeouts or jailing, search for `suspect` in the consensus logs for more information.

Crash logs from the child process are located at:
```
~/hl/data/visor_child_stderr/{date}/{node_binary_index}
```

---

## Validator Endpoints

Check the current validator summaries with:
```bash
curl -X POST --header "Content-Type: application/json" --data '{ "type": "validatorSummaries"}' https://api.hyperliquid-testnet.xyz/info
```
For **mainnet**, use the corresponding mainnet endpoint (if provided).

To change your validator profile (for example, updating the IP address):
```bash
~/hl-node --chain Testnet --key <validator-key> send-signed-action '{"type": "CValidatorAction", "changeProfile": {"node_ip": {"Ip": "1.2.3.4"}, "name": "..."}}'
```
For mainnet, replace `Testnet` with `Mainnet`.

Other profile options include:
- `disable_delegations`: Set to true to disable delegations.
- `commission_bps`: Specifies the percentage of staking rewards the validator takes (default is 10000, meaning all rewards go to the validator, and this cannot be increased).
- `signer`: Allows setting a hot address for signing consensus messages.

---

## Mainnet Non-Validator Seed Peers

For running a non-validator on **mainnet**, add at least one of these IP addresses to `~/override_gossip_config.json`:
```
operator_name,root_ips
ASXN,20.188.6.225
ASXN,74.226.182.22
B-Harvest,57.182.103.24
B-Harvest,3.115.170.40
Nansen,46.105.222.166
Nansen,91.134.41.52
Hypurrscan,57.180.50.253
```

---

### Troubleshooting

Crash logs from the child process are written to:
```
~/hl/data/visor_child_stderr/{date}/{node_binary_index}
```

---
