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


