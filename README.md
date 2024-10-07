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

**Clone project repository**
```
cd && rm -rf lava
git clone https://github.com/lavanet/lava
cd lava
git checkout v3.1.0
```

**Build binary**
```
export LAVA_BINARY=lavad
make install
```

**Prepare cosmovisor directories**
```
mkdir -p $HOME/.lava/cosmovisor/genesis/bin
sudo ln -s $HOME/.lava/cosmovisor/genesis $HOME/.lava/cosmovisor/current -f
sudo ln -s $HOME/.lava/cosmovisor/current/bin/lavad /usr/local/bin/lavad -f
```

# Move binary to cosmovisor directory
mv $(which lavad) $HOME/.lava/cosmovisor/genesis/bin

# Set node CLI configuration
lavad config chain-id lava-mainnet-1
lavad config keyring-backend file
lavad config node tcp://localhost:19957

# Initialize the node
lavad init "Your Node Name" --chain-id lava-mainnet-1

# Download genesis and addrbook files
curl -L https://snapshots.nodejumper.io/lava/genesis.json > $HOME/.lava/config/genesis.json
curl -L https://snapshots.nodejumper.io/lava/addrbook.json > $HOME/.lava/config/addrbook.json

# Set seeds
sed -i -e 's|^seeds *=.*|seeds = "e023c3892862744081360a99a2666e8111b196d3@lava-seed.y2.finance:26656,810b95bb95de712d4f30c2f62738bd976c016831@lava-seed.finteh.org:26656,cec848e7d4c5a7ae305b27cda133d213435c110f@seed-lava.ibs.team:16680,a5e5e2971721803e2297c870adacb234a724fc87@seed-lava.r93axnodes.cloud:8256,258f523c96efde50d5fe0a9faeea8a3e83be22ca@seed.lava-mainnet-1.lava.aviaone.com:10291,8542cd7e6bf9d260fef543bc49e59be5a3fa9074@seed.publicnode.com:26656,b9dfd3f222e65ae605efc29dc9e3faecdc3b71d0@lava.seed.stavr.tech:197,b85358e035343a3b15e77e1102857dcdaf70053b@seeds.bluestake.net:29956,ebc272824924ea1a27ea3183dd0b9ba713494f83@lava-mainnet-seed.autostake.com:27066,6086779c98b6864aea1ae595b96a412aa11ec6c7@lava.seed.stakevillage.net:17760"|' $HOME/.lava/config/config.toml

# Set minimum gas price
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.00002ulava"|' $HOME/.lava/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.lava/config/app.toml

# Change ports
sed -i -e "s%:1317%:19917%; s%:8080%:19980%; s%:9090%:19990%; s%:9091%:19991%; s%:8545%:19945%; s%:8546%:19946%; s%:6065%:19965%" $HOME/.lava/config/app.toml
sed -i -e "s%:26658%:19958%; s%:26657%:19957%; s%:6060%:19960%; s%:26656%:19956%; s%:26660%:19961%" $HOME/.lava/config/config.toml

# Download latest chain data snapshot
curl "https://snapshots.nodejumper.io/lava/lava_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.lava"

# Install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.6.0

# Create a service
sudo tee /etc/systemd/system/lava.service > /dev/null << EOF
[Unit]
Description=Lava node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.lava
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.lava"
Environment="DAEMON_NAME=lavad"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable lava.service

# Start the service and check the logs
sudo systemctl start lava.service
sudo journalctl -u lava.service -f --no-hostname -o cat
Secure Server Setup (Optional)

# generate ssh keys, if you don't have them already, DO IT ON YOUR LOCAL MACHINE
ssh-keygen -t rsa

# save the output, we'll use it later on instead of YOUR_PUBLIC_SSH_KEY
cat ~/.ssh/id_rsa.pub
# upgrade system packages
sudo apt update
sudo apt upgrade -y

# add new admin user
sudo adduser admin --disabled-password -q

# upload public ssh key, replace YOUR_PUBLIC_SSH_KEY with the key above
mkdir /home/admin/.ssh
echo "YOUR_PUBLIC_SSH_KEY" >> /home/admin/.ssh/authorized_keys
sudo chown admin: /home/admin/.ssh
sudo chown admin: /home/admin/.ssh/authorized_keys

echo "admin ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# disable root login, disable password authentication, use ssh keys only
sudo sed -i 's|^PermitRootLogin .*|PermitRootLogin no|' /etc/ssh/sshd_config
sudo sed -i 's|^ChallengeResponseAuthentication .*|ChallengeResponseAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PasswordAuthentication .*|PasswordAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PermitEmptyPasswords .*|PermitEmptyPasswords no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PubkeyAuthentication .*|PubkeyAuthentication yes|' /etc/ssh/sshd_config

sudo systemctl restart sshd

# install fail2ban
sudo apt install -y fail2ban

# install and configure firewall
sudo apt install -y ufw
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow 9100
sudo ufw allow 26656

# make sure you expose ALL necessary ports, only after that enable firewall
sudo ufw enable

# make terminal colorful
sudo su - admin
source <(curl -s https://raw.githubusercontent.com/nodejumper-org/cosmos-scripts/master/utils/enable_colorful_bash.sh)

# update servername, if needed, replace YOUR_SERVERNAME with wanted server name
sudo hostnamectl set-hostname YOUR_SERVERNAME

# now you can logout (exit) and login again using ssh admin@YOUR_SERVER_IP
