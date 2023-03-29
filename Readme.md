# \#draft / Деплой ноды Sepolia / Goerli
Структурно нода (ethereum client) состоит из двух элементов:
- Execution client (Geth, Nethermind, Besu, Erigon)
- Consensus client (Lighthouse, Lodestar, Nimbus, Prysm, Teku)

Рассмотрим простейший пример связки Geth + Lighthouse / Prysm. Клиент Goerli деплоится аналогичным образом после изменения некоторых параметров.

## Geth
---
Добавляем репо + ставим Geth

```
sudo apt install software-properties-common
sudo add-apt-repository -y ppa:ethereum/ethereum
sudo apt-get update
sudo apt-get install ethereum

geth version
```
Создаем сервис клиента Sepolia (синхронизация со снапа, для запуска полной ноды добавляем ключ `--syncmode full`. Информация о [ключах](https://geth.ethereum.org/docs/fundamentals/command-line-options))



```
sudo tee /etc/systemd/system/geth.service > /dev/null <<EOF
[Unit]
Description=Ethereum go client
After=network.target 
Wants=network.target

[Service]
User=root 
Group=root
Type=simple
Restart=always
RestartSec=5
ExecStart=/usr/bin/geth --sepolia --authrpc.addr localhost --authrpc.port 8551 --authrpc.vhosts localhost --http --http.addr 0.0.0.0 --http.port 9545 --http.api debug,eth,net,web3,txpool --ws --ws.addr 0.0.0.0 --ws.port 3335 --ws.api debug,eth,net,web3,txpool --ws.api eth,net,web3 --datadir /var/lib/goethereum --bootnodes "enode://9246d00bc8fd1742e5ad2428b80fc4dc45d786283e05ef6edbd9002cbc335d40998444732fbe921cb88e1d2c73d1b1de53bae6a2237    996e9bfe14f871baf7066@18.168.182.86:30303,enode://ec66ddcf1a974950bd4c782789a7e04f8aa7110a72569b6e65fcd51e937e74eed303b1ea734e4d19cfaec9fbff9b6ee65bf31dcb50ba79acce9dd63a6aca61c7@52.14.151.177:30303"

[Install]
WantedBy=default.target
EOF
```
Стартуем

```
sudo systemctl daemon-reload
sudo systemctl enable geth.service
sudo systemctl restart geth.service && journalctl -u geth.service -f -o cat
```
Далее на выбор рассмотрим два консенсус клиента Lighthouse и Prysm.

## Lighthouse
---
Качаем последний [релиз](https://github.com/sigp/lighthouse/releases)

```
wget https://github.com/sigp/lighthouse/releases/download/v4.0.1/lighthouse-v4.0.1-x86_64-unknown-linux-gnu.tar.gz

tar -C /usr/local/bin/ -vxzf lighthouse-v4.0.1-x86_64-unknown-linux-gnu.tar.gz

lighthouse --version
```
Сервис (синхронизация с чекпойнта, для запуска полной ноды убираем ключ `--checkpoint-sync-url`). [Документация](https://lighthouse-book.sigmaprime.io/)
```
sudo tee /etc/systemd/system/lighthoused.service > /dev/null <<EOF
[Unit]
Description=Lighthouse Beacon Node
After=network.target 
Wants=network.target

[Service]
User=root
Group=root
Type=simple
Restart=always
RestartSec=5
ExecStart=/usr/local/bin/lighthouse bn --network sepolia --enr-tcp-port 9010 --enr-udp-port 9010 --execution-endpoint http://localhost:8551 --execution-jwt /root/ethereum/geth/geth/jwtsecret --checkpoint-sync-url https://checkpoint-sync.sepolia.ethpandaops.io --disable-deposit-contract-sync

[Install]
WantedBy=default.target
EOF
```
Запускаем
```
sudo systemctl daemon-reload
sudo systemctl enable lighthoused.service
sudo systemctl restart lighthoused.service && journalctl -u lighthoused.service -f -o cat
```

В случае если на хосте **уже** имеется запущенный клиент Lighthouse (распространенная ситуация в докер клиентах) запуск второго **невозможен**. В этом случае следует использовать [альтернативный](https://ethereum.org/en/upgrades/get-involved/#clients) клиент. Рассмотрим в качестве такового [Prysm](https://docs.prylabs.network/docs/getting-started)

## Prysm
---
Ставим Prysm
```
cd $HOME
mkdir -p ethereum/consensus
mkdir ethereum/execution

cd ethereum/consensus
mkdir prysm && cd prysm---
curl https://raw.githubusercontent.com/prysmaticlabs/prysm/master/prysm.sh --output prysm.sh && chmod +x prysm.sh
```
Генерируем JWT Secret
```
./prysm.sh beacon-chain generate-auth-secret
```
Качаем [генезис](https://github.com/eth-clients/merge-testnets/blob/main/sepolia/genesis.ssz) Sepolia
```
wget -O genesis.ssz https://github.com/eth-clients/merge-testnets/blob/main/sepolia/genesis.ssz?raw=true
```
---
Строка запуска Geth. Обращаю внимание на необходимость изменения ключа `--authrpc` в сервисе `geth` созданном ранее.
```
geth --sepolia --http --http.api eth,net,engine,admin,web3 --authrpc.jwtsecret /root/ethereum/consensus/prysm/jwt.hex
```

Строка запуска Prysm. **Меняем ключ `--suggested-fee-recipient` на свой ERC20 адрес**
```
./prysm.sh beacon-chain --execution-endpoint=http://localhost:8551 --sepolia --suggested-fee-recipient=0x01234567722E6b0000012BFEBf6177F1D2e9758D9 --jwt-secret=jwt.hex --genesis-state=genesis.ssz
```
---

Закидываем скрипт в сервис (синхронизация с чекпойнта, для запуска полной ноды убираем ключ `--checkpoint-sync-url`). 
```
sudo tee /etc/systemd/system/prysm.service > /dev/null <<EOF
[Unit]
Description=Ethereum go client
After=network.target 
Wants=network.target

[Service]
User=root 
Group=root
Type=simple
Restart=always
RestartSec=5
ExecStart=/root/ethereum/consensus/prysm/prysm.sh beacon-chain --execution-endpoint=http://localhost:8551 --sepolia --checkpoint-sync-url=https://checkpoint-sync.sepolia.ethpandaops.io --suggested-fee-recipient=<0x56...06483> --jwt-secret=jwt.hex --genesis-state=genesis.ssz

[Install]
WantedBy=default.target
EOF
```
Стартуем
```
sudo systemctl daemon-reload
sudo systemctl enable prysm.service
sudo systemctl restart prysm.service && journalctl -u prysm.service -f -o cat
```

compiled by [road](https://t.me/ryssroad)