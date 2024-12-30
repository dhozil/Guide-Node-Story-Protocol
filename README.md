# Guide Node Story Protocol

## Testnet Setup Instructions

### Install dependencies

Update system and install build tools
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget build-essential jq make lz4 gcc unzip -y
```

### Install GO
```
cd $HOME && \
ver="1.22.3" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
source $HOME/.bash_profile && \
go version
```

### Init Moniker
```
echo "export MONIKER="yourmoniker"" >> $HOME/.bash_profile
echo "export STORY_CHAIN_ID="iliad-0"" >> $HOME/.bash_profile
echo "export STORY_PORT="52"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

### Download Story-Geth binary
```
cd $HOME
wget https://github.com/piplabs/story-geth/releases/download/v0.11.0/geth-linux-amd64
[ ! -d "$HOME/go/bin" ] && mkdir -p $HOME/go/bin
if ! grep -q "$HOME/go/bin" $HOME/.bash_profile; then
  echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
fi
chmod +x geth-linux-amd64
mv $HOME/geth-linux-amd64 $HOME/go/bin/story-geth
source $HOME/.bash_profile
story-geth version
```

### Download Story binary
```
cd
git clone https://github.com/piplabs/story && cd story
git checkout v0.13.0
go build -o story ./client
mv $HOME/story/story $HOME/go/bin/
source $HOME/.bash_profile
story version
```

### Create Geth Servie File
```
tee /etc/systemd/system/story-geth.service > /dev/null <<EOF
[Unit]
Description=Story Geth Client
After=network.target

[Service]
User=$USER
ExecStart=$HOME/go/bin/story-geth --odyssey --syncmode full --http --http.api eth,net,web3,engine --http.vhosts '*' --http.addr 127.0.0.1 --http.port 8545 --ws --ws.api eth,web3,net,txpool --ws.addr 127.0.0.1 --ws.port 8546
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

### Create Story Service File
```
tee /etc/systemd/system/story.service > /dev/null <<EOF
[Unit]
Description=Story Consensus Client
After=network.target

[Service]
User=$USER
WorkingDirectory=$HOME/.story/story
ExecStart=$HOME/go/bin/story run
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

## Cosmovisor Setup

### Install Cosmovisor
```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@latest
echo "export DAEMON_NAME="story"" >> $HOME/.bash_profile
echo "export DAEMON_HOME="$HOME/.story/story"" >> $HOME/.bash_profile
source $HOME/.bash_profile
cosmovisor init $(which story)
```

### Create a directory and download the current version
```
mkdir -p $HOME/.story/story/cosmovisor/upgrades/v0.13.0/bin
wget -O $HOME/.story/story/cosmovisor/upgrades/v0.13.0/bin/story https://github.com/piplabs/story/releases/download/v0.13.0/story-linux-amd64
chmod +x $HOME/.story/story/cosmovisor/upgrades/v0.13.0/bin/story
```

### Update service file
```
sudo tee /etc/systemd/system/story.service > /dev/null << EOF
[Unit]
Description=story node service
After=network-online.target

[Service]
User=$USER
Environment="DAEMON_NAME=story"
Environment="DAEMON_HOME=$HOME/.story/story"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_DATA_BACKUP_DIR=$HOME/.story/story/data"
ExecStart=$(which cosmovisor) run run
Restart=on-failure
RestartSec=10
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

### Reload and start story-geth
```
sudo systemctl daemon-reload && \
sudo systemctl start story-geth && \
sudo systemctl enable story-geth && \
sudo systemctl status story-geth
```

### Check logs
```
sudo journalctl -u story-geth -f -o cat
```
```
sudo journalctl -u story -f -o cat
```

### Check Sync status
```
curl localhost:26657/status | jq
```

### Create validator
#### View your validator key
```
story validator export
```

#### Export EVM private key
```
story validator export --export-evm-key
```

#### View EVM private key and make a key backup
```
cat $HOME/.story/story/config/private_key.txt
```

#### Create Your validator
```
story validator create --stake 1000000000000000000 --private-key "YOURPRIVATKEY" --moniker YOURMONIKER --chain-id 1516
```

## Delete Node
```
sudo systemctl stop story story-geth
sudo systemctl disable story story-geth
rm -rf $HOME/.story
sudo rm /etc/systemd/system/story.service /etc/systemd/system/story-geth.service
sudo systemctl daemon-reload
```
