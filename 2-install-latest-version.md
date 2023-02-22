# Install Cabal and GHC



First, update packages and install Ubuntu dependencies.

```bash
sudo apt-get update -y
```

```
sudo apt-get upgrade -y
```

```
sudo apt-get install automake build-essential pkg-config libffi-dev libgmp-dev libssl-dev libtinfo-dev libsystemd-dev zlib1g-dev make g++ tmux git jq wget libncursesw5 libtool autoconf liblmdb-dev llvm libnuma-dev libsodium-dev -y
```

Install Libsodium.

```bash
mkdir $HOME/git
cd $HOME/git
git clone https://github.com/input-output-hk/libsodium
cd libsodium
git checkout 66f017f1
./autogen.sh
./configure
make
sudo make install
```

extra lib linking may be required:
```bash
sudo ln -s /usr/local/lib/libsodium.so.23.3.0 /usr/lib/libsodium.so.23
```


Install Cabal and dependencies.

```bash
sudo apt-get -y install pkg-config libgmp-dev libssl-dev libtinfo-dev libsystemd-dev zlib1g-dev build-essential curl libgmp-dev libffi-dev libncurses-dev libtinfo5
```

```bash
curl --proto '=https' --tlsv1.2 -sSf https://get-ghcup.haskell.org | sh
```

Answer **NO **to installing haskell-language-server (HLS).

Answer **YES **to automatically add the required PATH variable to ".bashrc".

```bash
cd $HOME
source .bashrc
ghcup upgrade
ghcup install cabal 3.6.2.0
ghcup set cabal 3.6.2.0
```

Install GHC.

```bash
ghcup install ghc 8.10.7
ghcup set ghc 8.10.7
```

Update PATH to include Cabal and GHC and add exports. Your node's location will be in **$NODE\_HOME**. The [cluster configuration](https://hydra.iohk.io/job/Cardano/iohk-nix/cardano-deployment/latest-finished/download/1/index.html) is set by **$NODE\_CONFIG**.&#x20;

```bash
echo PATH="$HOME/.local/bin:$PATH" >> $HOME/.bashrc
echo export LD_LIBRARY_PATH="/usr/local/lib:$LD_LIBRARY_PATH" >> $HOME/.bashrc
echo export NODE_HOME=$HOME/cardano-my-node >> $HOME/.bashrc
echo export NODE_CONFIG=mainnet>> $HOME/.bashrc
source $HOME/.bashrc
```

Get and install secp256k1.

```bash
cd $HOME
git clone https://github.com/bitcoin-core/secp256k1.git
cd secp256k1
git reset --hard ac83be33d0956faf6b7f61a60ab524ef7d6a473a
./autogen.sh
./configure --prefix=/usr --enable-module-schnorrsig --enable-experimental
make
sudo make install
```



Update cabal and verify the correct versions were installed successfully.

```bash
cabal update
cabal --version
ghc --version
```

Cabal library should be version 3.6.2.0 and GHC should be version 8.10.7




# Build the node from source code

Download source code and switch to the latest tag.

```bash
cd $HOME/git
git clone https://github.com/input-output-hk/cardano-node.git
cd cardano-node
git fetch --all --recurse-submodules --tags
git checkout $(curl -s https://api.github.com/repos/input-output-hk/cardano-node/releases/latest | jq -r .tag_name)
```

Configure build options.

```
cabal update
cabal configure -O0 -w ghc-8.10.7
```

Update the cabal config, project settings, and reset build folder.

```bash
echo -e "package cardano-crypto-praos\n flags: -external-libsodium-vrf" > cabal.project.local
sed -i $HOME/.cabal/config -e "s/overwrite-policy:/overwrite-policy: always/g"
rm -rf $HOME/git/cardano-node/dist-newstyle/build/x86_64-linux/ghc-8.10.7
```

Build the cardano-node from source code.

```
cabal build cardano-cli cardano-node
```

Building process may take a few minutes up to a few hours depending on your computer's processing power.

Copy **cardano-cli ** and **cardano-node** files into bin directory.

```bash
sudo cp $(find $HOME/git/cardano-node/dist-newstyle/build -type f -name "cardano-cli") /usr/local/bin/cardano-cli
```

```bash
sudo cp $(find $HOME/git/cardano-node/dist-newstyle/build -type f -name "cardano-node") /usr/local/bin/cardano-node
```

Verify your **cardano-cli ** and **cardano-node** are the expected versions.

```
cardano-node version
cardano-cli version
```


## :triangular\_ruler: 3. Configure the nodes

Here you'll grab the config.json, genesis.json, and topology.json files needed to configure your node.

```bash
mkdir $NODE_HOME
cd $NODE_HOME
wget -N https://book.world.dev.cardano.org/environments/mainnet/config.json -O config.json
wget -N https://book.world.dev.cardano.org/environments/mainnet/db-sync-config.json -O db-sync-config.json
wget -N https://book.world.dev.cardano.org/environments/mainnet/submit-api-config.json -O submit-api-config.json
wget -N https://book.world.dev.cardano.org/environments/mainnet/byron-genesis.json -O byron-genesis.json
wget -N https://book.world.dev.cardano.org/environments/mainnet/shelley-genesis.json -O shelley-genesis.json
wget -N https://book.world.dev.cardano.org/environments/mainnet/alonzo-genesis.json -O alonzo-genesis.json
```

Run the following to modify **config.json** and&#x20;

* update TraceBlockFetchDecisions to "true"

```bash
sed -i config.json \
    -e "s/TraceBlockFetchDecisions\": false/TraceBlockFetchDecisions\": true/g"
```

Update **.bashrc **shell variables.

```bash
echo export CARDANO_NODE_SOCKET_PATH="$NODE_HOME/db/socket" >> $HOME/.bashrc
source $HOME/.bashrc
```


## :crystal\_ball: 4. Configure the block-producer node


A block producer node will be configured with various key-pairs needed for block generation (cold keys, KES hot keys and VRF hot keys). It can only connect to its relay nodes.


A relay node will not be in possession of any keys and will therefore be unable to produce blocks. It will be connected to its block-producing node, other relays and external nodes.


```bash
cat > $NODE_HOME/topology.json << EOF 
 {
    "Producers": [
      {
        "addr": "<RELAYNODE1'S IP ADDRESS>",
        "port": 6000,
        "valency": 1
      }
    ]
  }
EOF
```


## :flying\_saucer: 5. Configure the relay node(s)


On your other server that will be designed as your relay node or what we will call **relaynode1** throughout this guide, carefully **repeat steps 1 through 3** in order to build the cardano binaries.


You can have multiple relay nodes as you scale up your stake pool architecture. Simply create **relaynodeN** and adapt the guide instructions accordingly.


On your **relaynode1, **run** **with the following after updating with your block producer's IP address.

```bash
cat > $NODE_HOME/${NODE_CONFIG}-topology.json << EOF 
 {
    "Producers": [
      {
        "addr": "<BLOCK PRODUCER NODE'S IP ADDRESS>",
        "port": 6000,
        "valency": 1
      },
      {
        "addr": "relays-new.cardano-mainnet.iohk.io",
        "port": 3001,
        "valency": 2
      }
    ]
  }
EOF
```

## :lock\_with\_ink\_pen: 6. Configure the air-gapped offline machine


An air-gapped offline machine is called your cold environment.&#x20;

* Protects against key-logging attacks, malware/virus based attacks and other firewall or security exploits.&#x20;
* Physically isolated from the rest of your network.&#x20;
* Must not have a network connection, wired or wireless.&#x20;
* Is not a VM on a machine with a network connection.
* Learn more about [air-gapping at wikipedia](https://en.wikipedia.org/wiki/Air\_gap\_\(networking\)).



```bash
echo export NODE_HOME=$HOME/cardano-my-node >> $HOME/.bashrc
source $HOME/.bashrc
mkdir -p $NODE_HOME
```


Copy from your **hot environment**, also known as your block producer node, a copy of the **`cardano-cli` **to your **cold environment**, this air-gapped offline machine.&#x20;

Location of your cardano-cli.

```
/usr/local/bin/cardano-cli
```


In order to remain a true air-gapped environment, you must move files physically between your cold and hot environments with USB keys or other removable media.

After copying over to your cold environment, add execute permissions to the file.

```
sudo chmod +x /usr/local/bin/cardano-cli
```


## :robot: 7. Create startup scripts

The startup script contains all the variables needed to run a cardano-node such as directory, port, db path, config file, and topology file.

Run on Block Producer:
```bash
cat > $NODE_HOME/startBlockProducingNode.sh << EOF 
#!/bin/bash
DIRECTORY=$NODE_HOME
PORT=6000
HOSTADDR=0.0.0.0
TOPOLOGY=\${DIRECTORY}/topology.json
DB_PATH=\${DIRECTORY}/db
SOCKET_PATH=\${DIRECTORY}/db/socket
CONFIG=\${DIRECTORY}/config.json
/usr/local/bin/cardano-node run +RTS -N -RTS --topology \${TOPOLOGY} --database-path \${DB_PATH} --socket-path \${SOCKET_PATH} --host-addr \${HOSTADDR} --port \${PORT} --config \${CONFIG}
EOF
```

Run on RelayNode1:
```bash
cat > $NODE_HOME/startRelayNode1.sh << EOF 
#!/bin/bash
DIRECTORY=$NODE_HOME
PORT=6000
HOSTADDR=0.0.0.0
TOPOLOGY=\${DIRECTORY}/topology.json
DB_PATH=\${DIRECTORY}/db
SOCKET_PATH=\${DIRECTORY}/db/socket
CONFIG=\${DIRECTORY}/config.json
/usr/local/bin/cardano-node run +RTS -N -RTS --topology \${TOPOLOGY} --database-path \${DB_PATH} --socket-path \${SOCKET_PATH} --host-addr \${HOSTADDR} --port \${PORT} --config \${CONFIG}
EOF
```

Add execute permissions to the startup script.
```bash
chmod +x $NODE_HOME/startBlockProducingNode.sh
```

```bash
chmod +x $NODE_HOME/startRelayNode1.sh 
```


Run the following to create a **systemd unit file** to define your`cardano-node.service` configuration.


#### :cake: Benefits of using systemd for your stake pool

1. Auto-start your stake pool when the computer reboots due to maintenance, power outage, etc.
2. Automatically restart crashed stake pool processes.
3. Maximize your stake pool up-time and performance.


Run on Block Producer:
```bash
cat > $NODE_HOME/cardano-node.service << EOF 
# The Cardano node service (part of systemd)
# file: /etc/systemd/system/cardano-node.service 

[Unit]
Description     = Cardano node service
Wants           = network-online.target
After           = network-online.target 

[Service]
User            = ${USER}
Type            = simple
WorkingDirectory= ${NODE_HOME}
ExecStart       = /bin/bash -c '${NODE_HOME}/startBlockProducingNode.sh'
KillSignal=SIGINT
RestartKillSignal=SIGINT
TimeoutStopSec=300
LimitNOFILE=32768
Restart=always
RestartSec=5
SyslogIdentifier=cardano-node

[Install]
WantedBy	= multi-user.target
EOF
```


Run on RelayNode1:
```bash
cat > $NODE_HOME/cardano-node.service << EOF 
# The Cardano node service (part of systemd)
# file: /etc/systemd/system/cardano-node.service 

[Unit]
Description     = Cardano node service
Wants           = network-online.target
After           = network-online.target 

[Service]
User            = ${USER}
Type            = simple
WorkingDirectory= ${NODE_HOME}
ExecStart       = /bin/bash -c '${NODE_HOME}/startRelayNode1.sh'
KillSignal=SIGINT
RestartKillSignal=SIGINT
TimeoutStopSec=300
LimitNOFILE=32768
Restart=always
RestartSec=5
SyslogIdentifier=cardano-node

[Install]
WantedBy	= multi-user.target
EOF
```


Move the unit file to `/etc/systemd/system` and give it permissions.

```bash
sudo mv $NODE_HOME/cardano-node.service /etc/systemd/system/cardano-node.service
```

```bash
sudo chmod 644 /etc/systemd/system/cardano-node.service
```

Run the following to enable auto-starting of your stake pool at boot time.

```
sudo systemctl daemon-reload
sudo systemctl enable cardano-node
```

Your stake pool is now managed by the reliability and robustness of systemd. Below are some commands for using systemd.



#### :arrows\_counterclockwise: Restarting the node service

```
sudo systemctl reload-or-restart cardano-node
```

#### 🗄 Viewing and filter logs

```bash
journalctl --unit=cardano-node --follow
```


## :white\_check\_mark: 8. Start the nodes

Start your stake pool with systemctl and begin syncing the blockchain!


```bash
sudo systemctl start cardano-node
```

Install gLiveView, a monitoring tool.


gLiveView displays crucial node status information and works well with systemd services. Credits to the [Guild Operators](https://cardano-community.github.io/guild-operators/#/Scripts/gliveview) for creating this tool.


```bash
cd $NODE_HOME
sudo apt install bc tcptraceroute -y
curl -s -o gLiveView.sh https://raw.githubusercontent.com/cardano-community/guild-operators/master/scripts/cnode-helper-scripts/gLiveView.sh
curl -s -o env https://raw.githubusercontent.com/cardano-community/guild-operators/master/scripts/cnode-helper-scripts/env
chmod 755 gLiveView.sh
```

Run the following to modify **env** with the updated file locations.

```bash
sed -i env \
    -e "s/\#CONFIG=\"\${CNODE_HOME}\/files\/config.json\"/CONFIG=\"\${NODE_HOME}\/mainnet-config.json\"/g" \
    -e "s/\#SOCKET=\"\${CNODE_HOME}\/sockets\/node0.socket\"/SOCKET=\"\${NODE_HOME}\/db\/socket\"/g"
```

A node must reach epoch 208 (Shelley launch) before **gLiveView.sh** can start tracking the node syncing. You can track the node syncing using `journalctl `before epoch 208.

```
journalctl --unit=cardano-node --follow
```

Run gLiveView to monitor the progress of the sync'ing of the blockchain.

```
./gLiveView.sh
```
