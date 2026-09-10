[🇬🇧 English](README.en.md) | 🇹🇷 Türkçe

# Worrell Testnet Node Kurulumu

Resmi kaynaklar:
- Node kaynak kodu: https://github.com/worrellchain/worrell
- Network/genesis reposu: https://github.com/worrellchain/networks
- Ayrıntılı validator rehberi: `docs/RUNNING-A-NODE.md` (worrellchain/worrell reposunda)

| Alan | Değer |
|---|---|
| Chain ID | `worrell-testnet-1` |
| Binary | `worrelld` (Cosmos SDK v0.53.6) |
| Versiyon | `v0.1.2` |
| Genesis SHA256 | `a81c507b12ba0678c3172394ff4bb03e1c3db60050cc5568c127a24ec19378fd` |
| Persistent peer (resmi) | `bb9164c1bd9ed9ff2c0fd9e09b23285698e231de@164.68.98.186:26656` |
| Persistent peer (ITRocket) | `40128ea31b1cfb5d4b24fc9e32ee0c468586c983@worrell-testnet-peer.itrocket.net:12656` |
| Min gas price | `0.025uworrell` |
| Faucet | `POST http://164.68.98.186:4500` `{"address":"worrell1..."}` (500 WORRELL) |

## Donanım gereksinimleri

- 4+ fiziksel CPU çekirdeği
- En az 200 GB SSD disk
- En az 8 GB RAM
- En az 100 Mbps ağ bant genişliği

> **Not:** Aynı sunucuda başka Cosmos SDK zincirleri (ör. symphonyd, titand, pchaind)
> çalıştırıyorsanız varsayılan portlar (26656/26657/1317/9090) çakışabilir.
> Bu rehberdeki port değiştirme adımı (9. adım) bunun için var.

## 1. Bağımlılıkları kur

```bash
sudo apt update && sudo apt upgrade -y && sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make ncdu gcc git jq chrony liblz4-tool -y
sudo apt install zip -y
```

## 2. Go kur

Go 1.24.5 kullanılıyor. Zaten kuruluysa bu adımı atlayabilirsiniz.

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

## 3. Binary kur

Kaynaktan derleyerek:

```bash
cd $HOME
rm -rf worrell
git clone https://github.com/worrellchain/worrell.git
cd worrell
git checkout v0.1.2
make install
```

Kurulumu doğrula:

```bash
worrelld version --long | grep -e commit -e version
```

> Alternatif: chain registry'de her platform için önceden derlenmiş binary
> linkleri de var (`worrellchain/networks/worrell-testnet-1/chain.json` içinde,
> `codebase.binaries` alanı) — kaynaktan derlemek istemiyorsanız oradan indirebilirsiniz.

## 4. Node'u başlat (init)

`<Moniker>` yerine kendi node isminizi yazın.

```bash
worrelld init <Moniker> --chain-id worrell-testnet-1
```

## 5. Genesis indir (resmi kaynak)

```bash
curl -s https://raw.githubusercontent.com/worrellchain/networks/main/worrell-testnet-1/genesis.json \
  -o ~/.worrell/config/genesis.json
```

Checksum ile doğrula:

```bash
echo "a81c507b12ba0678c3172394ff4bb03e1c3db60050cc5568c127a24ec19378fd  $HOME/.worrell/config/genesis.json" | sha256sum -c -
```

`OK` çıktısı almazsanız dosyayı kullanmayın, tekrar indirin.

## 6. Gas ve peer ayarları

```bash
worrelld config set config p2p.persistent_peers "bb9164c1bd9ed9ff2c0fd9e09b23285698e231de@164.68.98.186:26656,40128ea31b1cfb5d4b24fc9e32ee0c468586c983@worrell-testnet-peer.itrocket.net:12656"
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.025uworrell\"/;" ~/.worrell/config/app.toml
```

## 7. Pruning ayarları

```bash
pruning="custom"
pruning_keep_recent="100"
pruning_interval="20"
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.worrell/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.worrell/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.worrell/config/app.toml
```

## 8. Indexer'ı kapat

```bash
sed -i 's|^indexer *=.*|indexer = "null"|' $HOME/.worrell/config/config.toml
```

## 9. Portları değiştir (diğer chain'lerle çakışma varsa)

Sunucuda başka Cosmos node'ları çalışıyorsa önce boş portları tespit edin:

```bash
for p in 26666 26667 26617 26690; do echo -n "$p: "; sudo lsof -i :$p >/dev/null 2>&1 && echo DOLU || echo BOS; done
```

Boş çıkan portlara göre ayarlayın (örnek değerler):

```bash
sed -i 's/laddr = "tcp:\/\/127.0.0.1:26657"/laddr = "tcp:\/\/127.0.0.1:26667"/' ~/.worrell/config/config.toml
sed -i 's/laddr = "tcp:\/\/0.0.0.0:26656"/laddr = "tcp:\/\/0.0.0.0:26666"/' ~/.worrell/config/config.toml
sed -i 's/address = "tcp:\/\/0.0.0.0:1317"/address = "tcp:\/\/0.0.0.0:26617"/' ~/.worrell/config/app.toml
sed -i 's/address = "0.0.0.0:9090"/address = "0.0.0.0:26690"/' ~/.worrell/config/app.toml
```

Bu rehberin geri kalanında **RPC portu olarak 26667** kullanılmaktadır — kendi ortamınıza göre değiştirin.

## 10. systemd servisi oluştur ve başlat

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

P2P portunu değiştirdiyseniz firewall'da da açın:

```bash
sudo ufw allow 26666/tcp
```

## 11. Logları ve senkronizasyonu izle

```bash
sudo journalctl -u worrelld -f -o cat
```

Senkronizasyon durumu:

```bash
curl -s http://127.0.0.1:26667/status | jq '.result.sync_info.catching_up'
```

`false` dönene kadar node tam senkronize olmamış demektir. Node tam senkron olmadan
validator oluşturma adımına geçmeyin.

Hızlı senkron için resmi RPC uç noktaları da mevcut (statesync için kullanılabilir):
- `https://worrel-testnet-rpc.oshvank.xyz` (OshVanK)
- `https://worrell-testnet-rpc.itrocket.net` (ITRocket)

## 12. Cüzdan oluştur veya mevcut cüzdanı içe aktar

**Yeni cüzdan oluşturmak için:**

```bash
worrelld keys add validator-key --keyring-backend test
```

**Mevcut bir mnemonic'i içe aktarmak için:**

```bash
worrelld keys add validator-key --recover --keyring-backend test
```

Komut mnemonic'inizi soracak — sadece bu interaktif prompt'a yazın, asla komut
parametresi veya dosya olarak düz metin bırakmayın.

Adresi doğrulayın:

```bash
worrelld keys show validator-key -a --keyring-backend test
```

> Sunucu (headless) ortamda `--keyring-backend os` yerine `test` backend kullanmak
> genelde daha pratiktir. `test` backend disk'te şifrelenmemiş saklar; sadece
> testnet için kabul edilebilir, mainnet'te kullanmayın.

## 13. Faucet'ten test token al

```bash
curl -X POST http://164.68.98.186:4500 -H 'Content-Type: application/json' -d '{"address":"<adresiniz>"}'
```

Bakiyeyi kontrol edin:

```bash
worrelld query bank balances <adresiniz> --node http://127.0.0.1:26667
```

## 14. Validator oluştur

Node **tam senkron** olduktan sonra devam edin.

Consensus pubkey'i alın:

```bash
worrelld tendermint show-validator
```

`validator.json` dosyası oluşturun (pubkey ve moniker'ı kendinize göre değiştirin):

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

İşlemi gönderin:

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

Doğrulama:

```bash
worrelld query staking validator $(worrelld keys show validator-key --bech val -a --keyring-backend test) --node http://127.0.0.1:26667
```

`status: BOND_STATUS_BONDED` görmelisiniz.

## 15. Ek delegasyon (self-stake artırma)

```bash
worrelld tx staking delegate <worrelvaloper-adresiniz> <miktar>uworrell \
  --from validator-key \
  --chain-id worrell-testnet-1 \
  --gas auto \
  --gas-adjustment 1.4 \
  --gas-prices 0.025uworrell \
  --keyring-backend test \
  --node http://127.0.0.1:26667
```

Gas ücreti için bakiyenizde her zaman birkaç WORRELL'lik pay bırakın, tamamını
delege etmeyin.

## Güvenlik notları

- `priv_validator_key.json`, `node_key.json` ve cüzdan mnemonic'lerini asla
  paylaşmayın, repoya commit etmeyin veya sohbet/log gibi kalıcı ortamlara
  yapıştırmayın.
- Bu bir **testnet** kurulumudur (`worrell-testnet-1`), mainnet değildir —
  buradaki anahtarları/mnemonic'leri gerçek değeri olan hiçbir yerde
  kullanmayın.
- `test` keyring-backend disk'te şifrelenmemiş anahtar saklar; production/mainnet
  ortamında kullanmayın.
- Genesis dosyasını her zaman resmi kaynaktan indirin ve SHA256 checksum ile
  doğrulayın (5. adım) — üçüncü taraf sitelerden indirilen dosyalara güvenmeyin.
