**Hardware Requirements**

Minimum
```
4CPU 8RAM 100GB
```
Recommended
```
8CPU 32RAM 200GB
```

Rent On Hetzner | Rent On OVH
Dependencies Installation

**Install dependencies for building from source**
```
sudo apt update
sudo apt install -y curl git jq lz4 build-essential
```

**Install Go**
```
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.22.7.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.profile
source .profile
```

Node Installation

**Clone project repository**
```
cd && rm -rf noisd
git clone https://github.com/noislabs/noisd
cd noisd
git checkout v1.0.5
```

**Build binary**
```
make install
```

**Prepare cosmovisor directories**
```
mkdir -p $HOME/.noisd/cosmovisor/genesis/bin
ln -s $HOME/.noisd/cosmovisor/genesis $HOME/.noisd/cosmovisor/current -f
```

**Copy binary to cosmovisor directory**
```
cp $(which noisd) $HOME/.noisd/cosmovisor/genesis/bin
```

**Set node CLI configuration**
```
noisd config chain-id nois-1
noisd config keyring-backend file
noisd config node tcp://localhost:17357
```

**Initialize the node**
```
noisd init "Your Node Name" --chain-id nois-1
```

**Download genesis and addrbook files**
```
curl -L https://snapshots.nodejumper.io/nois/genesis.json > $HOME/.noisd/config/genesis.json
curl -L https://snapshots.nodejumper.io/nois/addrbook.json > $HOME/.noisd/config/addrbook.json
```

**Set seeds**
```
sed -i -e 's|^seeds *=.*|seeds = "b3e3bd436ee34c39055a4c9946a02feec232988c@seeds.cros-nest.com:56656,babc3f3f7804933265ec9c40ad94f4da8e9e0017@seed.rhinostake.com:17356,ade4d8bc8cbe014af6ebdf3cb7b1e9ad36f412c0@seeds.polkachu.com:17356,72cd4222818d25da5206092c3efc2c0dd0ec34fe@161.97.96.91:36656,20e1000e88125698264454a884812746c2eb4807@seeds.lavenderfive.com:17356,c8db99691545545402a1c45fa897f3cb1a05aea6@nois-mainnet-seed.itrocket.net:36656,1de5c83c5a5eb223c814401f0506b44b742741da@nois.peer.stavr.tech:40136,8542cd7e6bf9d260fef543bc49e59be5a3fa9074@seed.publicnode.com:26656"|' $HOME/.noisd/config/config.toml
```

**Set minimum gas price**
```
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.05unois"|' $HOME/.noisd/config/app.toml
```

**Set pruning**
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.noisd/config/app.toml
```

**Enable prometheus**
```
sed -i -e 's|^prometheus *=.*|prometheus = true|' $HOME/.noisd/config/config.toml
```

**Change ports**
```
sed -i -e "s%:1317%:17317%; s%:8080%:17380%; s%:9090%:17390%; s%:9091%:17391%; s%:8545%:17345%; s%:8546%:17346%; s%:6065%:17365%" $HOME/.noisd/config/app.toml
sed -i -e "s%:26658%:17358%; s%:26657%:17357%; s%:6060%:17360%; s%:26656%:17356%; s%:26660%:17361%" $HOME/.noisd/config/config.toml
```

**Download latest chain data snapshot**
```
curl "https://snapshots.nodejumper.io/nois/nois_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.noisd"
```

**Install Cosmovisor**
```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.6.0
```

**Create a service**
```
sudo tee /etc/systemd/system/nois.service > /dev/null << EOF
[Unit]
Description=Nois node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.noisd
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.noisd"
Environment="DAEMON_NAME=noisd"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable nois.service
```

**Start the service and check the logs**
```
sudo systemctl start nois.service
sudo journalctl -u nois.service -f --no-hostname -o cat
Secure Server Setup (Optional)
```

**generate ssh keys, if you don't have them already, DO IT ON YOUR LOCAL MACHINE**
```
ssh-keygen -t rsa
```

**save the output, we'll use it later on instead of YOUR_PUBLIC_SSH_KEY**
```
cat ~/.ssh/id_rsa.pub
```

**upgrade system packages**
```
sudo apt update
sudo apt upgrade -y
```

**add new admin user**
```
sudo adduser admin --disabled-password -q
```

**upload public ssh key, replace YOUR_PUBLIC_SSH_KEY with the key above**
```
mkdir /home/admin/.ssh
echo "YOUR_PUBLIC_SSH_KEY" >> /home/admin/.ssh/authorized_keys
sudo chown admin: /home/admin/.ssh
sudo chown admin: /home/admin/.ssh/authorized_keys

echo "admin ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
```

**disable root login, disable password authentication, use ssh keys only**
```
sudo sed -i 's|^PermitRootLogin .*|PermitRootLogin no|' /etc/ssh/sshd_config
sudo sed -i 's|^ChallengeResponseAuthentication .*|ChallengeResponseAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PasswordAuthentication .*|PasswordAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PermitEmptyPasswords .*|PermitEmptyPasswords no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PubkeyAuthentication .*|PubkeyAuthentication yes|' /etc/ssh/sshd_config

sudo systemctl restart sshd
```
**install fail2ban**
```
sudo apt install -y fail2ban
```

**install and configure firewall**
```
sudo apt install -y ufw
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow 9100
sudo ufw allow 26656
```

**make sure you expose ALL necessary ports, only after that enable firewall**
```
sudo ufw enable
```

**make terminal colorful**
```
sudo su - admin
source <(curl -s https://raw.githubusercontent.com/nodejumper-org/cosmos-scripts/master/utils/enable_colorful_bash.sh)
```

**update servername, if needed, replace YOUR_SERVERNAME with wanted server name**
```
sudo hostnamectl set-hostname YOUR_SERVERNAME
```

**now you can logout (exit) and login again using ssh admin@YOUR_SERVER_IP**

Add project overview
Add feature summary
Add system requirements
Add setup instructions
Add quick start guide
Document data ingestion process
Add visualization examples
Add performance tuning tips
Add API reference
