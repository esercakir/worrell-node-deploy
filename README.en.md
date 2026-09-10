Worrell Testnet Node Setup
Official sources:
Node source code: https://github.com/worrellchain/worrell
Network/genesis repo: https://github.com/worrellchain/networks
Full validator guide: `docs/RUNNING-A-NODE.md` (in the worrellchain/worrell repo)
Field	Value
Chain ID	`worrell-testnet-1`
Binary	`worrelld` (Cosmos SDK v0.53.6)
Version	`v0.1.2`
Genesis SHA256	`a81c507b12ba0678c3172394ff4bb03e1c3db60050cc5568c127a24ec19378fd`
Persistent peer (official)	`bb9164c1bd9ed9ff2c0fd9e09b23285698e231de@164.68.98.186:26656`
Persistent peer (ITRocket)	`40128ea31b1cfb5d4b24fc9e32ee0c468586c983@worrell-testnet-peer.itrocket.net:12656`
Min gas price	`0.025uworrell`
Faucet	`POST http://164.68.98.186:4500` `{"address":"worrell1..."}` (500 WORRELL)
Hardware requirements
4+ physical CPU cores
At least 200 GB SSD
At least 8 GB RAM
At least 100 Mbps network bandwidth
> **Note:** If you're running other Cosmos SDK chains (e.g. symphonyd, titand,
> pchaind) on the same server, the default ports (26656/26657/1317/9090) may
> collide. Step 9 below covers changing ports to avoid this.
1. Install dependencies
```bash
sudo apt update && sudo apt upgrade -y && sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make ncdu gcc git jq chrony liblz4-tool -y
sudo apt install zip -y
```
2. Install Go
Uses Go 1.24.5. Skip this step if already installed.
```bash
ver="1.24.5"
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=\$PATH:/usr/local/go/bin:\$HOME/go/bin" >> ~/.bash_profile
source ~/.bash_profile
go version
```
3. Build the binary
From source:
```bash
cd $HOME
rm -rf worrell
git clone https://github.com/worrellchain/worrell.git
cd worrell
git checkout v0.1.2
make install
```
Verify:
```bash
worrelld version --long | grep -e commit -e version
```
> Alternative: the chain registry also lists prebuilt binaries for each
> platform (`worrellchain/networks/worrell-testnet-1/chain.json`, under
> `codebase.binaries`) if you'd rather not build from source.
4. Initialize the node
Replace `<Moniker>` with your own node name.
```bash
worrelld init <Moniker> --chain-id worrell-testnet-1
```
5. Download genesis (official source)
```bash
curl -s https://raw.githubusercontent.com/worrellchain/networks/main/worrell-testnet-1/genesis.json \
  -o ~/.worrell/config/genesis.json
```
Verify the checksum:
```bash
echo "a81c507b12ba0678c3172394ff4bb03e1c3db60050cc5568c127a24ec19378fd  $HOME/.worrell/config/genesis.json" | sha256sum -c -
```
If you don't get `OK`, don't use the file — re-download it.
6. Gas and peer settings
```bash
worrelld config set config p2p.persistent_peers "bb9164c1bd9ed9ff2c0fd9e09b23285698e231de@164.68.98.186:26656,40128ea31b1cfb5d4b24fc9e32ee0c468586c983@worrell-testnet-peer.itrocket.net:12656"
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.025uworrell\"/;" ~/.worrell/config/app.toml
```
7. Pruning settings
```bash
pruning="custom"
pruning_keep_recent="100"
pruning_interval="20"
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.worrell/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.worrell/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.worrell/config/app.toml
```
8. Disable the indexer
```bash
sed -i 's|^indexer *=.*|indexer = "null"|' $HOME/.worrell/config/config.toml
```
9. Change ports (if colliding with other chains)
If you're running other Cosmos nodes on the same server, first find free ports:
```bash
for p in 26666 26667 26617 26690; do echo -n "$p: "; sudo lsof -i :$p >/dev/null 2>&1 && echo IN_USE || echo FREE; done
```
Set them according to what's free (example values):
```bash
sed -i 's/laddr = "tcp:\/\/127.0.0.1:26657"/laddr = "tcp:\/\/127.0.0.1:26667"/' ~/.worrell/config/config.toml
sed -i 's/laddr = "tcp:\/\/0.0.0.0:26656"/laddr = "tcp:\/\/0.0.0.0:26666"/' ~/.worrell/config/config.toml
sed -i 's/address = "tcp:\/\/localhost:1317"/address = "tcp:\/\/0.0.0.0:26617"/' ~/.worrell/config/app.toml
sed -i 's/address = "localhost:9090"/address = "0.0.0.0:26690"/' ~/.worrell/config/app.toml
```
> Double-check the *exact* current value in your config files before running
> `sed` — defaults can be `localhost:PORT` rather than `0.0.0.0:PORT` or
> `127.0.0.1:PORT` depending on the binary version, and a mismatched pattern
> will silently do nothing.
This guide uses RPC port 26667 from here on — adjust to your own setup.
Also check `pprof_laddr` in `config.toml` (`[instrumentation]` section) if you
hit a "port already in use" error for `localhost:6060` on startup — it's a
non-fatal error but worth moving to a free port too.
10. Create and start the systemd service
```bash
sudo tee /etc/systemd/system/worrelld.service > /dev/null <<EOF
[Unit]
Description=Worrell
After=network-online.target
[Service]
User=$USER
ExecStart=$(which worrelld) start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable worrelld
sudo systemctl restart worrelld
```
If you changed the P2P port, open it in the firewall too:
```bash
sudo ufw allow 26666/tcp
```
11. Watch logs and sync status
```bash
sudo journalctl -u worrelld -f -o cat
```
Sync status:
```bash
curl -s http://127.0.0.1:26667/status | jq '.result.sync_info.catching_up'
```
Wait for this to return `false` before creating a validator. Also make sure
the RPC listen address is bound to `0.0.0.0`, not `127.0.0.1`, if any
external service (e.g. an explorer/monitoring tool on another server) needs
to query it — check `config.toml`'s `[rpc]` `laddr` value.
Official RPC endpoints are also available for faster sync (statesync):
`https://worrel-testnet-rpc.oshvank.xyz` (OshVanK)
`https://worrell-testnet-rpc.itrocket.net` (ITRocket)
12. Create a wallet or import an existing one
To create a new wallet:
```bash
worrelld keys add validator-key --keyring-backend test
```
To import an existing mnemonic:
```bash
worrelld keys add validator-key --recover --keyring-backend test
```
The command will prompt for your mnemonic — type it only into this
interactive prompt, never as a command-line argument, into a file, or into
any chat/log that gets persisted.
Verify the address:
```bash
worrelld keys show validator-key -a --keyring-backend test
```
> On a headless server, using the `test` keyring backend is usually more
> practical than `os`. The `test` backend stores keys unencrypted on disk —
> acceptable for testnet only, never use it for mainnet.
13. Get testnet tokens from the faucet
```bash
curl -X POST http://164.68.98.186:4500 -H 'Content-Type: application/json' -d '{"address":"<your-address>"}'
```
Check the balance:
```bash
worrelld query bank balances <your-address> --node http://127.0.0.1:26667
```
14. Create a validator
Only proceed once the node is fully synced.
Get the consensus pubkey:
```bash
worrelld tendermint show-validator
```
Create a `validator.json` file (replace pubkey and moniker with your own):
```bash
cat > validator.json << 'EOF'
{
  "pubkey": {"@type":"/cosmos.crypto.ed25519.PubKey","key":"<pubkey>"},
  "amount": "100000000uworrell",
  "moniker": "<moniker>",
  "identity": "",
  "website": "",
  "security": "",
  "details": "",
  "commission-rate": "0.10",
  "commission-max-rate": "0.20",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1"
}
EOF
```
Submit the transaction:
```bash
worrelld tx staking create-validator validator.json \
  --from validator-key \
  --chain-id worrell-testnet-1 \
  --gas auto \
  --gas-adjustment 1.4 \
  --gas-prices 0.025uworrell \
  --keyring-backend test \
  --node http://127.0.0.1:26667
```
Verify:
```bash
worrelld query staking validator $(worrelld keys show validator-key --bech val -a --keyring-backend test) --node http://127.0.0.1:26667
```
You should see `status: BOND_STATUS_BONDED`.
15. Additional delegation (increase self-stake)
```bash
worrelld tx staking delegate <your-valoper-address> <amount>uworrell \
  --from validator-key \
  --chain-id worrell-testnet-1 \
  --gas auto \
  --gas-adjustment 1.4 \
  --gas-prices 0.025uworrell \
  --keyring-backend test \
  --node http://127.0.0.1:26667
```
Always leave a small balance for gas fees — don't delegate 100% of your
balance.
Security notes
Never share, commit to a repo, or paste into chat/logs your
`priv_validator_key.json`, `node_key.json`, or wallet mnemonic.
This is a testnet setup (`worrell-testnet-1`), not mainnet — never
reuse these keys/mnemonics anywhere with real value.
The `test` keyring backend stores keys unencrypted; don't use it in
production/mainnet.
Always download the genesis file from the official source and verify its
SHA256 checksum (step 5) — don't trust genesis/addrbook files from
third-party sites.
