

Newsflash: There is a brand new `install.sh` which I want you to try out. 
It saves you a lot of manual work, because it is replacing the whole first half of all of this. 
Just go to the main folder `cd chainhammer/` and from there(!) execute: `scripts/install.sh`. 
But: No guarantees. 
Better only use on a disposable/cloud/virtualbox machine, and NOT on your main work machine!!

# chainhammer
Actually, today I tried this again - tested on and optimized for Debian AWS machine (`debian-stretch-hvm-x86_64-gp2-2018-08-20-85640`) - all this really does work:

## TOC
* How to replicate the results  
  * toolchain: docker & -compose, solc
  * parity-deploy
  * chainhammer install (repo, virtualenv, dependencies)
  * chainhammer config
  * chainhammer start
* check an individual transactions for success/failure
  * geth docker - how to?
  * geth install via golang
* geth clique network (for comparison with above parity network results)
* AWS deployment - how I created the AMI
* [readymade Amazon AMI](https://github.com/drandreaskrueger/chainhammer/blob/master/reproduce.md#readymade-amazon-ami) <-- start HERE if you have little time -->
  * how to clone your own AWS machine from that image
  * how to benchmark `parity`
  * how to benchmark `geth`
  * chainhammer:
    * chainhammer: test connection
    * chainhammer: TPS watcher
    * chainhammer: send 20,000 transactions
* AWS --> measurement results 
  * [numbers](#results) <-- jump here if you have only 1 minute -->
  * configurations used
* issues raised while doing this


## How to replicate the results

### VPS machine 
#### swap - in case the RAM is not sufficient:
```
SWAPFILE=/swapfile; sudo dd if=/dev/zero of=$SWAPFILE bs=1M count=700 && sudo chmod 600 $SWAPFILE && sudo mkswap $SWAPFILE && echo $SWAPFILE none swap defaults 0 0 | sudo tee -a /etc/fstab && sudo swapon -a && free -m
```
(for quorum-crux use not 700 but count=1500, see below)

#### useful tools
```
sudo apt-get update && sudo apt-get -y upgrade
sudo apt-get install -y wget htop jq
```

#### N.B.: before creating image from instance to make a new AMI
[remove-ssh-host-key-pairs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/building-shared-amis.html?icmpid=docs_ec2_console#remove-ssh-host-key-pairs), update chainhammer repo to newest commit, then power down:
```
sudo shred -u /etc/ssh/*_key /etc/ssh/*_key.pub
cd ~/drandreaskrueger_chainhammer; git pull
sudo shutdown now
```


### toolchain
```
# docker
# this is for Debian Linux, 
# if you run a different distro, google "install docker [distro name]"
sudo apt-get update 
sudo apt-get -y remove docker docker-engine docker.io 
sudo apt-get install -y apt-transport-https ca-certificates wget software-properties-common
wget https://download.docker.com/linux/debian/gpg 
sudo apt-key add gpg
rm gpg
echo "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee -a /etc/apt/sources.list.d/docker.list
sudo apt-get update
sudo apt-cache policy docker-ce
sudo apt-get -y install docker-ce 
sudo systemctl start docker

sudo usermod -aG docker ${USER}
groups $USER
```
log out and log back in, to enable those usergroup changes

```
# docker compose new version
sudo curl -L "https://github.com/docker/compose/releases/download/1.22.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod 755 /usr/local/bin/docker-compose
docker-compose --version
docker --version
```
> docker-compose version 1.22.0, build f46880fe  
> docker version 18.06.1-ce, build e68fc7a  

```
# solc
# someone should PLEASE create a Debian specific installation routine
# see https://solidity.readthedocs.io/en/latest/installing-solidity.html 
# and https://github.com/ethereum/solidity/releases
wget https://github.com/ethereum/solidity/releases/download/v0.4.24/solc-static-linux
chmod 755 solc-static-linux 
echo $PATH
sudo mv solc-static-linux /usr/local/bin/
sudo ln -s /usr/local/bin/solc-static-linux /usr/local/bin/solc
solc --version
```
> Version: 0.4.24+commit.e67f0147.Linux.g++


### dockerized parity network
```
# parity-deploy.sh
# for a dockerized parity environment
# this is instantseal, NOT a realistic network of nodes
# but it already shows the problem - parity is very slow.
# For a more realistic network of 4 aura nodes see chainhammer-->parity.md
git clone https://github.com/paritytech/parity-deploy.git paritytech_parity-deploy
cd paritytech_parity-deploy
sudo ./clean.sh
./parity-deploy.sh --config dev --name instantseal --geth
docker-compose up
```
That starts one `instantseal` node - but that already shows that parity cannot get faster than ~70 TPS. Use this for now.

(Later, you can try `aura` networks of 4 nodes instead - see instructions 
here: [parity.md --> run 13](../results/parity.md#run-13) )


### chainhammer
new terminal:
```
# chainhammer & dependencies
git clone https://github.com/drandreaskrueger/chainhammer drandreaskrueger_chainhammer
cd drandreaskrueger_chainhammer/

sudo apt install python3-pip libssl-dev
sudo pip3 install virtualenv 
virtualenv -p python3 py3eth
source env/bin/activate

python3 -m pip install --upgrade pip==18.0

pip3 install --upgrade py-solc==3.1.0 web3==4.7.2 web3[tester]==4.7.2 rlp==0.6.0 eth-testrpc==1.3.5 requests==2.19.1 pandas==0.23.4 matplotlib==3.0.0 pytest pytest-cov
```

or if you are adventurous:
```
virtualenv -p python3 env/py3eth
source env/env/bin/activate
python3 -m pip install --upgrade pip==18.0
pip3 install --upgrade rlp==0.6.0 eth-testrpc py-solc web3 web3[tester] requests pandas matplotlib pytest pytest-cov
```
The rlp==0.6.0 is reported https://github.com/pipermerriam/eth-testrpc/issues/118

```
# configure chainhammer
nano config.py

RPCaddress, RPCaddress2 = 'http://localhost:8545', 'http://localhost:8545'
ROUTE = "web3"
```

```
# test connection
touch account-passphrase.txt
./deploy.py 
```

```
# start the chainhammer viewer
./tps.py
```

new terminal


```
# same virtualenv
cd drandreaskrueger_chainhammer
source env/bin/activate

# start the chainhammer send routine
./deploy.py notest; ./send.py 
```

or:

```
# not blocking but with 23 multi-threading workers
./deploy.py notest; ./send.py threaded2 23
```
---

### everything below here is *not necessary*
... to improve parity's speed.

(it just provides a local `geth` installation, for `geth attach http://localhost:8545`)

new terminal

( * )

```
# check that the transactions are actually successfully executed:

geth attach http://localhost:8545

> web3.eth.getTransaction(web3.eth.getBlock(50)["transactions"][0])
{
  gas: 90000, ...
}

> web3.eth.getTransactionReceipt(web3.eth.getBlock(50)["transactions"][0])
{ 
  gasUsed: 26691,
  status: "0x1", ...
}
> 
```

### geth - dockerized: please help

( * ) I actually do *not* want to install `geth` locally, but start the geth console *from a docker container* - but no success yet:

```
docker run ethereum/client-go attach https://localhost:8545
```
> WARN [09-10|09:38:24.984] Sanitizing cache to Go's GC limits       provided=1024 updated=331  
> Fatal: Failed to start the JavaScript console: api modules: Post https://localhost:8545: dial tcp 127.0.0.1:8545: connect: connection refused  
> Fatal: Failed to start the JavaScript console: api modules: Post https://localhost:8545: dial tcp 127.0.0.1:8545: connect: connection refused  

Please help me with ^ this, thanks.

...

Until that is sorted, I simply install `geth` *locally*:

```
wget https://dl.google.com/go/go1.11.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.11.linux-amd64.tar.gz 
rm go1.11.linux-amd64.tar.gz 
echo "export PATH=\$PATH:/usr/local/go/bin:~/go/bin" >> .profile
```
logout, log back in
```
go version
```
> go version go1.11 linux/amd64

```
go get -d github.com/ethereum/go-ethereum
go install github.com/ethereum/go-ethereum/cmd/geth
geth version
```
> geth version  
> WARN [09-10|09:56:11.759] Sanitizing cache to Go's GC limits       provided=1024 updated=331  
> Geth  
> Version: 1.8.16-unstable  
> Architecture: amd64  
> Protocol Versions: [63 62]  
> Network Id: 1  
> Go Version: go1.11  
> Operating System: linux  
> GOPATH=  
> GOROOT=/usr/local/go  


## geth clique
Compare the poor TPS performance of `parity aura` with the faster `geth clique`:

#### stop parity
Kill the above `parity-deploy.sh ...; docker-compose up` with:

Ctrl-C, then

```
docker-compose down -v
```
you might run out of disk space, so better delete any docker stuff:
```
docker kill $(docker ps -q) ; docker rm $(docker ps -a -q) ; docker rmi $(docker images -q)
```

#### geth Clique network with dockerized nodes
for details see [geth.md#javahippiegeth-dev](https://github.com/drandreaskrueger/chainhammer/blob/0bdcbedfeeb261c534ae3baeb0bd9a37054c9b28/geth.md#javahippiegeth-dev).

```
git clone https://github.com/drandreaskrueger/geth-dev.git drandreaskrueger_geth-dev
cd drandreaskrueger_geth-dev

docker-compose up
```
wait until you see healthy looking logging, like

```
monitor-frontend         | 2018-09-10 13:34:02.054 [API] [BLK] Block: 7 from: gethDev1
```

##### start benchmark

new terminal: test connection
```
cd drandreaskrueger_chainhammer/
source env/bin/activate
./deploy.py
```


new terminal: watcher
```
cd drandreaskrueger_chainhammer/
source env/bin/activate
./tps.py
```

new terminal: hammer
```
cd drandreaskrueger_chainhammer/
source env/bin/activate
./deploy.py notest; ./send.py 
```


## quorum-crux-IBFT

A dockerized quorum-crux network of 4 nodes using IBFT (*Istanbul Byzantine Fault Tolerance*) consensus:

### local build

Edit the given `blk-io/crux`, so that it builds a local docker container, instead of using `blkio10/quorum-crux:v1.0.0` from dockerhub:

```
cd ~
git clone  https://github.com/blk-io/crux blk-io_crux
git checkout f39db2345cf9d82e76d3905468e6e5ea1469b09d

cd blk-io_crux/docker/quorum-crux/
cp docker-compose.yaml docker-compose-local.yaml

nano docker-compose-local.yaml 
```
Follow the instructions "to build the Docker images yourself", so that it looks like this:
```
...
  node1: &quorum_crux_node
    # Pull image down from Docker Hub
    # image: blkio10/quorum-crux:v1.0.0
    # Uncomment the below, and comment out the above line to build the Docker images yourself
    image: blk.io/quorum/quorum-crux
    build:
      context: .
    container_name: quorum1
...
```

### TPS-optimized network settings 

Edit the settings: Higher gas limit, larger txpool, blockperiod 1 second
```
sudo apt install jq
jq '.gasLimit = "0x1312D00"' istanbul-genesis.json > tmp && mv tmp istanbul-genesis.json

sed -i 's/PRIVATE_CONFIG/ARGS=$ARGS"--txpool.globalslots 20000 --txpool.globalqueue 20000 --istanbul.blockperiod 1 "\nPRIVATE_CONFIG/g' istanbul-start.sh 
```

### build and start
then build the container, and start the network:
```
docker-compose -f docker-compose-local.yaml up --build
```

wait until you see something like

```
...
quorum1  | [*] Starting Crux nodes
quorum3  | [*] Starting Ethereum nodes
...
quorum1  | set +v
```

## AWS deployment

This first part here you can safely ignore, it just logs what I have done to create the AMI:

### how I created the AMI
* [Launch instance Wizard](https://eu-west-2.console.aws.amazon.com/ec2/v2/home?region=eu-west-2#LaunchInstanceWizard:) in  `eu-west-2` (London)
* Community AMIs, search "Debian-stretch 2018"
* newest is `debian-stretch-hvm-x86_64-gp2-2018-08-20-85640`
  * ami-0e5de816bb166040f
  * FAI Debian image
  * Root device type: ebs 
  * Virtualization type: hvm`
* type `t2.micro`
* Step 3: Configure Instance Details
  * Network: Default
  * Subnet: Default in eu-west-2a
  * auto assign public IP
* Step 5: Add Tags
  * Name: chainhammer
  * Environment: dev
  * Project: benchmarking
  * Owner: Andreas Krueger
* create new security group, name it; allow ssh access
* choose an existing ssh keypair `AndreasKeypairAWS.pem`

simplify `ssh` access, by adding this block to your local machine's

```
nano ~/.ssh/config
```

```
Host chainhammer
  Hostname ec2-35-178-181-232.eu-west-2.compute.amazonaws.com
  StrictHostKeyChecking no
  User admin
  IdentityFile ~/.ssh/AndreasKeypairAWS.pem
```
now it becomes this simple to connect:
```
ssh chainhammer
```
Then in that machine I created a small swap to protect against lack of memory:

```
SWAPFILE=/swapfile
sudo dd if=/dev/zero of=$SWAPFILE bs=1M count=512 &&
sudo chmod 600 $SWAPFILE &&
sudo mkswap $SWAPFILE &&

echo $SWAPFILE none swap defaults 0 0 | sudo tee -a /etc/fstab &&
sudo swapon -a &&
free -m
```

**Then I executed all the above instructions (install the toolchain, and chainhammer).**

Then after removing all docker containers & images (to save space)
```
~/remove-all-docker.sh
```
I shutdown the instance, and created an AMI from it, and made it public.

```
sudo shutdown now
```
On [AWS console #Instances](https://eu-west-2.console.aws.amazon.com/ec2/v2/home?region=eu-west-2#Instances) ... actions ... create image.

On [AWS console #Images](https://eu-west-2.console.aws.amazon.com/ec2/v2/home?region=eu-west-2#Images) ... right click ... Modify Image Permissions ... public. And tag it, like above.

--> AMI ID `ami-0aaa64f3e432e4a26`

By now that AMI is superseded. Use the "search for public AMIs --> chainhammer" instead, next chapter:

## readymade Amazon AMI 

This will much accelerate your own benchmarking experiments. In my ready-made Amazon AWS image I have done all of the above 
(Plus some unlogged updates, of the toolchain & chainhammer).

Use my AMI:

* In the Public Images, [search for "chainhammer"](https://eu-west-2.console.aws.amazon.com/ec2/v2/home?region=eu-west-2#Images:visibility=public-images;search=chainhammer;sort=name) in `eu-west-2` (London).
* Right click ... Launch
* Step 2: Choose an Instance Type --> at least `t2.small` ! otherwise you probably run out of memory
* Step 4: Add Storage --> `12 GB` !
* security group, choose an existing ssh keypair e.g. `AndreasKeypairAWS.pem` (obviously, your keypair instead)
* launch, wait

simplify the `ssh` access, by adding this (with *your* IP, obviously) to your local machine's
```
nano ~/.ssh/config
```
```
Host chainhammer
  Hostname ec2-35-178-181-232.eu-west-2.compute.amazonaws.com
  StrictHostKeyChecking no
  User admin
  IdentityFile ~/.ssh/AndreasKeypairAWS.pem
```
now it becomes this simple to connect:
```
ssh chainhammer
```


### parity
```
ssh chainhammer

cd ~/paritytech_parity-deploy
# sed -i 's/0x1312D00/0x2625A00/g' config/spec/genesis/aura; cat config/spec/genesis/aura # hardcoded now in parity-deploy https://github.com/paritytech/parity-deploy/issues/55#issuecomment-422309365

ARGS="--db-compaction ssd --tracing off --gasprice 0 --gas-floor-target 100000000000 "
ARGS=$ARGS"--pruning fast --tx-queue-size 32768 --tx-queue-mem-limit 0 --no-warp "
ARGS=$ARGS"--jsonrpc-threads 8 --no-hardware-wallets --no-dapps --no-secretstore-http "
ARGS=$ARGS"--cache-size 4096 --scale-verifiers --num-verifiers 16 --force-sealing "

sudo ./clean.sh 
./parity-deploy.sh --nodes 4 --config aura --name myaura --geth $ARGS

sed -i 's/parity:stable/parity:v1.11.11/g' docker-compose.yml
jq ".engine.authorityRound.params.stepDuration = 5" deployment/chain/spec.json > tmp; mv tmp deployment/chain/spec.json

cp ~/paritytech_parity-deploy/deployment/1/password ~/drandreaskrueger_chainhammer/account-passphrase.txt
docker-compose up
```
For explanations of all those settings, see [parity.md](../results/parity.md). 

If you later want to end this ... 'Ctrl-c' and:

```
docker-compose down -v
sudo ./clean.sh
```

or even
```
~/remove-all-docker.sh docker-compose -f docker-compose-local.yaml up --build
```

### geth

```
ssh chainhammer

cd ~/drandreaskrueger_geth-dev/
docker-compose up
```

If you want to end this ... 'Ctrl-c' and:

```
docker-compose down -v
```

### quorum-crux IBFT

```
ssh chainhammer
```

note the `-local` in the docker-compose command! -->
```
cd ~/blk-io_crux/docker/quorum-crux/
docker-compose -f docker-compose-local.yaml up --build
```

If you want to end this ... 'Ctrl-c' and:

```
docker-compose -f docker-compose-local.yaml down -v
```

and possibly destroy all docker
```
~/remove-all-docker.sh
```

N.B.: Port is changed from 8545 to 22001:

```
nano ~/drandreaskrueger_chainhammer/config.py
```
change to
```
RPCaddress, RPCaddress2 = 'http://localhost:22001', 'http://localhost:22002'
```
##### careful
Until `blk-io/crux` have also included a [docker-compose-local.yaml](#local-build) into their own repo, pay attention whether the two .yaml are still in sync:
```
diff docker-compose-local.yaml docker-compose.yaml
```

### chainhammer: test connection
... and create some local files
```
ssh chainhammer
cd ~/drandreaskrueger_chainhammer && source env/bin/activate

./deploy.py
```
If there are connection problems, probably need to configure the correct ports 
in [config.py](../hammer/config.py):
```
nano config.py
```
or set the correct account passphrase:
```
cp ~/paritytech_parity-deploy/deployment/1/password ~/drandreaskrueger_chainhammer/account-passphrase.txt
```


### chainhammer: watcher
```
./tps.py
```

### chainhammer: send transactions
```
ssh chainhammer
cd drandreaskrueger_chainhammer && source env/bin/activate

./deploy.py notest; ./send.py threaded2 23
```

## results

| hardware  	| node type 	    | #nodes 	| config 	| peak TPS_av 	| final TPS_av 	|
|-----------	|-----------	    |--------	|--------	|-------------	|--------------	|
| t2.micro 	    | parity aura   	| 4      	| (D)    	| 45.5        	|  44.3        |
| t2.large 	    | parity aura   	| 4      	| (D)    	| 53.5        	|  52.9        |
| t2.xlarge 	| parity aura   	| 4      	| (J)    	| 57.1        	|  56.4        |
| t2.2xlarge 	| parity aura   	| 4      	| (D)    	| 57.6        	|  57.6        |
|               |                   |           |        	|         	    |              |
| t2.micro 	    | parity instantseal | 1      	| (G)    	| 42.3        	|  42.3        |
| t2.xlarge	    | parity instantseal | 1      	| (J)    	| 48.1        	|  48.1        |
|               |                   |           |        	|         	    |              |
| t2.2xlarge 	| geth clique     	| 3+1 +2    | (B)    	| 421.6       	| 400.0        |
| t2.xlarge 	| geth clique     	| 3+1 +2    | (B)    	| 386.1       	| 321.5        |
| t2.xlarge 	| geth clique     	| 3+1       | (K)    	| 372.6       	| 325.3        |
| t2.large 	    | geth clique     	| 3+1 +2    | (B)    	| 170.7       	| 169.4        |
| t2.small 	    | geth clique     	| 3+1 +2    | (B)    	|  96.8       	|  96.5        |
| t2.micro 	    | geth clique     	| 3+1       | (H)    	| 124.3       	| 122.4        |
|               |                   |           |        	|         	    |              |
| t2.micro SWAP | quorum crux IBFT 	| 4    	    | (I) SWAP! |  98.1         |  98.1   	   |
|               |                   |           |        	|         	    |              |
| t2.micro 	    | quorum crux IBFT 	| 4    	    | (F)     	| lack of RAM   |         	   |
| t2.large 	    | quorum crux IBFT 	| 4    	    | (F)    	| 207.7      	| 199.9        |
| t2.xlarge 	| quorum crux IBFT 	| 4    	    | (F)    	| 439.5      	| 395.7        |
| t2.xlarge 	| quorum crux IBFT 	| 4    	    | (L)    	| 389.1      	| 338.9        |
| t2.2xlarge 	| quorum crux IBFT 	| 4    	    | (F)    	| 435.4      	| 423.1        |
| c5.4xlarge 	| quorum crux IBFT 	| 4    	    | (F)    	| 536.4      	| 524.3        |

For the hardware types, number of CPUs etc - see https://aws.amazon.com/ec2/instance-types/t2/#Product_Details

Only `t2.micro` is *"free tier"*, i.e. please contact me, if you can support me financially, so I can keep testing this on larger machines.

We need completely new ideas how to accelerate parity.

### (A) parity aura  
4 nodes via [paritytech/parity-deploy](https://github.com/paritytech/parity-deploy) with higher gasLimit and gasFloorTarget, and some CLI parameters changed (*you knowledgable parity experts, please experiment with those, to increase the TPS - thanks*):
```
cd ~/paritytech_parity-deploy
# sed -i 's/0x1312D00/0x2625A00/g' config/spec/genesis/aura; cat config/spec/genesis/aura # hardcoded now in parity-deploy https://github.com/paritytech/parity-deploy/issues/55#issuecomment-422309365

./parity-deploy.sh --nodes 4 --config aura --name myaura --geth --jsonrpc-server-threads 10 --tx-queue-size 20000 --cache-size 4096 --gas-floor-target 40000000 --tx-queue-mem-limit 0
cp ~/paritytech_parity-deploy/deployment/1/password ~/drandreaskrueger_chainhammer/account-passphrase.txt
docker-compose up
```
> Parity/v1.11.11-stable-cb03f38-20180910/x86_64-linux-gnu/rustc1.28.0

### (B) geth clique
1 bootnode, and 3 miners nodes, and ethstats client and server, all dockerized. 

Two parameters changed: [gasLimit=40m and clique.period=2 seconds](https://github.com/drandreaskrueger/geth-dev/commit/e1fe5ab4c464e406e3898cbe0152d3c1aa4c6697).
```
cd ~/drandreaskrueger_geth-dev/
docker-compose up
```
> Geth/v1.8.14-stable-316fc7ec/linux-amd64/go1.10.3

### (C) geth quorum IBFT
TODO, some unsolved issues. What worked fine on my local machine does not seem to work anymore on AWS. Strange.

### (D) parity v1.11.11 because 2.0.5 is broken

Unfortunately, they have named `2.0.5` now into `stable` prematurely, so that `docker run parity/parity:stable` starts `parity:v2.0.x` and not `parity:v1.11.x` anymore. 

That *broke everything*, see [parity.md --> run 11](../results/parity.md). 

To correct that, replace `:stable` with the old version `:v1.11.11` *after* running `parity-deploy.sh` (I had also made a [feature request issue](https://github.com/paritytech/parity-deploy/issues/55) about this):
```
cd paritytech_parity-deploy
sudo ./clean.sh
docker kill $(docker ps -q); docker rm $(docker ps -a -q); docker rmi $(docker images -q)

ARGS="--db-compaction ssd --tracing off --gasprice 0 --gas-floor-target 100000000000 "
ARGS=$ARGS"--pruning fast --tx-queue-size 32768 --tx-queue-mem-limit 0 --no-warp "
ARGS=$ARGS"--jsonrpc-threads 8 --no-hardware-wallets --no-dapps --no-secretstore-http "
ARGS=$ARGS"--cache-size 4096 --scale-verifiers --num-verifiers 16 "

./parity-deploy.sh --nodes 4 --config aura --name myaura --geth $ARGS

sed -i 's/parity:stable/parity:v1.11.11/g' docker-compose.yml

docker-compose up
```

### (E) include all current recommendations of parity team:

Use (D) plus `--force-sealing` plus change the blocktime `stepDuration` to 5 seconds, because [5chdn said so](https://github.com/paritytech/parity-ethereum/issues/9586#issuecomment-422717091):

```
cd ~/paritytech_parity-deploy
sudo ./clean.sh

docker kill $(docker ps -q); docker rm $(docker ps -a -q); docker rmi $(docker images -q)

ARGS="--db-compaction ssd --tracing off --gasprice 0 --gas-floor-target 100000000000 "
ARGS=$ARGS"--pruning fast --tx-queue-size 32768 --tx-queue-mem-limit 0 --no-warp "
ARGS=$ARGS"--jsonrpc-threads 8 --no-hardware-wallets --no-dapps --no-secretstore-http "
ARGS=$ARGS"--cache-size 4096 --scale-verifiers --num-verifiers 16 --force-sealing "

./parity-deploy.sh --nodes 4 --config aura --name myaura --geth $ARGS

sed -i 's/parity:stable/parity:v1.11.11/g' docker-compose.yml
jq ".engine.authorityRound.params.stepDuration = 5" deployment/chain/spec.json > tmp; mv tmp deployment/chain/spec.json

docker-compose up
```
This ^ (E) is the newest set of suggested settings, but they actually do not accelerate over the results of the already measured settings (D).

### (F) quorum IBFT settings

Standard dockerized quorum-crux but with a local build, so that these parameters can be tuned before the docker containers are built:

```
gasLimit = "0x1312D00"

--txpool.globalslots 20000 
--txpool.globalqueue 20000 
--istanbul.blockperiod 1
```

See above [#quorum-crux-ibft](#quorum-crux-ibft) for how to do that.

Tried the same with increasing machine sizes, up to 16 vCPUs. Best result 524-536 TPS.

### (G) parity instantseal
```
cd ~/paritytech_parity-deploy
sudo ./clean.sh
docker kill $(docker ps -q); docker rm $(docker ps -a -q); docker rmi $(docker images -q)

./parity-deploy.sh --config dev --name instantseal --geth 

sed -i 's/parity:stable/parity:v1.11.11/g' docker-compose.yml

docker-compose up
```

#### interesting observation:
The blocking version of `send.py` is actually a bit *faster* than the multi-threaded, i.e. hammering with `./deploy.py notest; ./send.py` (instead of `./deploy.py notest; ./send.py threaded2 23`) results in the fastest TPS. 

Is parity essentially single-threaded? 

Also, the go client `geth` benefits greatly from larger machines, i.e. more CPUs; but `parity` shows only very mildly faster TPS on larger machines.

### (H) geth on t2.micro
to make it work on the AWS "free tier" machine, I removed the ethstats docker "geth-monitor-front/backend" - see issue [GD#33](https://github.com/javahippie/geth-dev/issues/33):

```
cd ~/drandreaskrueger_geth-dev
nano docker-compose-without-ethstats.yml
docker-compose -f docker-compose-without-ethstats.yml up --build
```

and even on that small machine I could see well over 100 TPS with geth clique!

### (I) quorum-crux on t2.micro with swapping
To run this 4 nodes dockerized blk-io/crux setup is more difficult because each node runs an instance of geth_quorum AND an instance of crux. I have already posted a [feature request BC#48](https://github.com/blk-io/crux/issues/48) = it would be nice to still be able to run this on a t2.micro all in RAM. For now, you can enlarge the swapfile:
```
sudo swapoff -a && SWAPFILE=/swapfile; sudo dd if=/dev/zero of=$SWAPFILE bs=1M count=1500 && sudo chmod 600 $SWAPFILE && sudo mkswap $SWAPFILE && sudo swapon -a && free -m
```
and keep an instance of `htop` open to notice when the ceiling is hit (the you get connection problems because node 1 or 2 has run out of memory, and crashed):
```
ssh chainhammer
htop
```

other than that, this is identical to (F) above:

```
cd ~/blk-io_crux/docker/quorum-crux
docker-compose -f docker-compose-local.yaml up --build
```

**beware that these results are artifically slow** because swapping not RAM. But I could get it running on an AWS `t2.micro` which is "free tier" - so you can reproduce it without paying!!!

### (J) parity v1.11.11 on AWS t2.xlarge
Repeated recent run, to get *chainreader diagrams* for parity on a fast AWS machine.  
See [parity.md#run-18](../results/parity.md#run-18) for details. Almost identical to (D) above, just newer versions of some dependencies.

### (K) geth v1.8.14 on AWS t2.xlarge
Repeated recent run, to get *chainreader diagrams* for geth clique on a fast AWS machine. See [geth.md#run-2](geth.md#run-2) for details. Almost identical to (B), but without the ethstats docker instances.

Interesting new observation, now that I ran it for 50k transactions not 20k. See issue comment  https://github.com/ethereum/go-ethereum/issues/17447#issuecomment-431629285


### (L) IBFT quorum-crux on AWS t2.xlarge
Repeated recent run, to get *chainreader diagrams* for quorum IBFT on a fast AWS machine. See [quorum-IBFT.md#run-11](quorum-IBFT.md#run-11) for details. Almost identical to (F), so I don't know why it is suddenly 15% slower. 

Perhaps it is a newer version of quorum? Unfortunately I don't know as [quorum pretends to be geth](https://github.com/jpmorganchase/quorum/issues/507), and is stuck on version `Geth/v1.7.2` for a very long time now.


## you
Please inspire us what could make `parity aura` faster. 
Or actually ... what could make *any* of this faster. Thanks.

## issues
See bottom of [parity.md](../results/parity.md#issues), [geth.md](../results/geth.md#issues), 
[quorum.md](../results/quorum.md#issues-raised), [quorum-IBFT.md](../results/quorum-IBFT.md#issues).

