# Run an AVAX node and validatoron on a RaspberryPi Everest edition

## Setup

- Rapsberry Pi 4B 8GB RAM (2 and 4GB RAM should work, too)
- Micro SD card 128GB SanDisk Ultra (64GB should be enough for now)
- official Power Supply USB-C DC 5.1V 3A
- network cable
- Armor Case passive cooling
- someday I might need an external SSD but not yet

## Install Ubuntu 20.4

First we need to install a 64bit system. Raspian is a 32bit system so we can not use it.
Best choice is Ubuntu.  
[Download Ubuntu 20.04 LTS 64-bit for RasPi4](https://ubuntu.com/download/raspberry-pi)  
If you don't know how to get the image on the SD-card follow the [tutorial](https://ubuntu.com/tutorials/how-to-install-ubuntu-on-your-raspberry-pi#1-overview).

## Set up ssh

Because we don't want to plug in a keyboard and screen all the time we will use ssh to connect to the RasPi. Therefore we have do create a file called `ssh` in the boot partion. Following this [tutorial](https://linuxhint.com/raspberry_pi_headless_mode_ubuntu/).

Now we can connect us to the Raspi from any computer in the same network with `ssh ubuntu@ubuntu`. The initial password is `ubuntu`.  
Securing your RasPi, e.g. using keys for ssh or unattended-upgrades, is recommended but not part of this tutorial.
You should update your RasPi with `sudo apt update` followed by `sudo apt full-upgrade`.

## Add some extra GB Memory aka swap

Especially when you use a RasPi with less than 3GB RAM you should give the system some swap, that is a kind of extra RAM on your SD-card. With the 8GB RasPi you can skip this part.

I used this [tutorial](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-18-04) to add 8GB. Normaly you should make it x1 to x2 the size of your RAM.

```sh
sudo fallocate -l 8G /swapfile #8G is the size here 8GB
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudo cp /etc/fstab /etc/fstab.bak #just in case a backup
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## byobu

```sh
sudo apt install byobu #should already be installed
```

byobu is the perfect shell if you want to use stuff like `nohub`, `screen` or `tmux` but the easy-way.
It is the perfect shell when you connected via ssh, as it runs in the background even when your connection breaks down.

From here on I **ALWAYS** work in **byobu** on my RasPi. When I get disconnected the process keeps on running and when I come back and type `byobu` I am where I left, only exception is when the RasPi was rebooted in the meantime.

After you connect to your RasPi just type `byobu`. Here are some [tips](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-byobu-for-terminal-management-on-ubuntu-16-04) how to use it.

## Create an user for gecko

```sh
sudo adduser --gecos ',,,' --disabled-password --home /opt/gecko gecko
```

Whenever we need this user we get it with `sudo su gecko`. The user itself has no password and can not login. So we login as `ubuntu` and switch to `gecko` when needed.

The $HOME folder for this user is `/opt/gecko`.

**For the following steps I will use the user `gecko` with `sudo su gecko`.**

## Install AVA/gecko

I used this [tutorial](https://medium.com/@nitaxmx/ava-testnet-validator-in-raspberry-pi-59bd16b59d64).

First we install the language go with `sudo apt install golang`.

Check the version with `go version` we need go1.13.x or higher.

We continue with some stuff taken from [ava-labs github](https://github.com/ava-labs/gecko#installation).

### $GOPATH

The GOPATH should already be set ([golang github](https://github.com/golang/go/wiki/SettingGOPATH)) but somehow **it is not**.  
We still use the `gecko` user.  
Try `echo $GOPATH` if you see a blank line use the following commands to set the GOPATH.

- `cd ~`
- `mkdir go`
- `nano .bashrc`
- add `export GOPATH=$HOME/go` and an empty line to the end of the file
- save the file and close nano
- `source .bashrc`
- `echo $GOPATH` should return `/opt/gecko/go`

### gecko

Still user `gecko` still using `byobu`.

Let's build gecko from source.

```sh
sudo apt-get install libssl-dev libuv1-dev cmake make curl g++
go get -v -d github.com/ava-labs/gecko/...
cd $GOPATH/src/github.com/ava-labs/gecko
./scripts/build.sh
```

## Join the network

I follow mostly this [article](https://medium.com/avalabs/how-to-join-avalanche-everest-release-a40cd20aa654).

```sh
cd $GOPATH/src/github.com/ava-labs/gecko/build
./avalanche
```

While gecko is running, keep it running and open another window `F2` in byobu (`F3` and `F4` to switch windows, `exit` to close, `F6` to detach byobu you can come back anytime just type `byobu`)

In the new window we make an API call to create an user.

### Create an user

```sh
curl -X POST --data '{
     "jsonrpc": "2.0",
     "id": 1,
     "method": "keystore.createUser",
     "params": {
         "username": "YOUR USERNAME",
         "password": "YOUR PASSWORD"
     }
}' -H 'content-type:application/json;' 127.0.0.1:9650/ext/keystore
```

You will get a response like this:

```sh
response {"jsonrpc":"2.0","result":{"success":true},"id":1}
```

[Documentation](https://docs.avax.network/v1.0/en/api/keystore/#keystorecreateuser)

NOW WE WAIT AND GET FRUSTRATED - DON'T ... GET FRUSTRATED BUT BE PATIENT

Your node is bootstrapping and this may take a little while up to a few hours with bad luck. But this will only take so long on your very first start or when the node has been offline a long time.  

What you will see after you entered `./avalanche`  
Some ASCII-Art and output

Ignore the NAT-traversal warning for the time beeing [FAQ](https://docs.avax.network/v1.0/en/faq/faq/#node-prints-nat-traversal-failed)

How do you know your node finished bootsptrapping? There are two API calls for that.

```sh
curl -X POST --data '{
    "jsonrpc":"2.0",
    "id"     :1,
    "method" :"health.getLiveness"
}' -H 'content-type:application/json;' 127.0.0.1:9650/ext/health
```

Response when all is bootstrapped, otherwise you will see a human readable error.

```sh
{
  "jsonrpc": "2.0",
  "result": {
    "checks": {
      "chains.default.bootstrapped": {
        "timestamp": "2020-08-26T19:17:54.123530254Z",
        "duration": 16982,
        "contiguousFailures": 0,
        "timeOfFirstFailure": null
      },
      "network.validators.heartbeat": {
        "message": {
          "heartbeat": 1598469474
        },
        "timestamp": "2020-08-26T19:17:54.123466939Z",
        "duration": 10185,
        "contiguousFailures": 0,
        "timeOfFirstFailure": null
      }
    },
    "healthy": true
  },
  "id": 1
}
```

[Documentation](https://docs.avax.network/v1.0/en/api/health/#healthgetliveness)

Or you can check every chain with this call, set the chain in the params.

```sh
curl -X POST --data '{
    "jsonrpc":"2.0",
    "id"     :1,
    "method" :"info.isBootstrapped",
    "params": {
        "chain":"X"
    }
}' -H 'content-type:application/json;' 127.0.0.1:9650/ext/info
```

[Documentation](https://docs.avax.network/v1.0/en/api/info/#infoisbootstrapped)

### Create an address on the X-Chain

```sh
curl -X POST --data '{
    "jsonrpc":"2.0",
    "id"     :2,
    "method" :"avm.createAddress",
    "params" :{
        "username": "YOUR USERNAME",
        "password": "YOUR PASSWORD"
    }
}' -H 'content-type:application/json;' 127.0.0.1:9650/ext/bc/X
```

Response is something like this:

```sh
{
    "jsonrpc": "2.0",
    "result": {
        "address": "YOUR X-CHAIN ADDRESS"
    },
    "id": 1
}
```

[Documentation](https://docs.avax.network/v1.0/en/api/avm/#avmcreateaddress)

Write down your address.

But you can request it with:

```sh
curl -X POST --data '{
    "jsonrpc": "2.0",
    "method": "avm.listAddresses",
    "params": {
        "username":"myUsername",
        "password":"myPassword"
    },
    "id": 1
}' -H 'content-type:application/json;' 127.0.0.1:9650/ext/bc/X
```

[Documentation](https://docs.avax.network/v1.0/en/api/avm/#avmlistaddresses)

Now you should get yourself some AVAX from [faucet](https://faucet.avax.network/). Use your X-address.

### Create an address on the P-Chain

```sh
curl -X POST --data '{
    "jsonrpc": "2.0",
    "method": "platform.createAddress",
    "params": {
        "username": "YOUR USERNAME",
        "password": "YOUR PASSWORD"
    },
    "id": 1
}' -H 'content-type:application/json;' 127.0.0.1:9650/ext/P
```

Response:

```sh
{
    "jsonrpc": "2.0",
    "result": {
        "address": "YOUR P-CHAIN ADDRESS"
    },
    "id": 1
}
```

[Documentation](https://docs.avax.network/v1.0/en/api/platform/#platformcreateaddress)

Write it down.

But you can request it with:

```sh
curl -X POST --data '{
    "jsonrpc":"2.0",
    "id"     :1,
    "method" :"platform.listAddresses",
    "params" :{
        "username": "YOUR USERNAME",
        "password": "YOUR PASSWORD"
    }
}' -H 'content-type:application/json;' 127.0.0.1:9650/ext/P
```

[Documentation](https://docs.avax.network/v1.0/en/api/platform/#platformlistaddresses)

## Send AVA to the Platform P-Chain


```sh
curl -X POST --data '{
    "jsonrpc":"2.0",
    "id"     :1,
    "method" :"avm.exportAVAX",
    "params" :{
         "username": "YOUR USERNAME",
         "password": "YOUR PASSWORD",
        "to":"YOUR P-CHAIN ADDRESS",
        "amount": 7000000
    }
}' -H 'content-type:application/json;' 127.0.0.1:9650/ext/bc/X
```

[Documentation](https://docs.avax.network/v1.0/en/api/platform/#platformexportavax)

You don't need the returned txID.

### Accept the tranfer on P-Chain

```sh
curl -X POST --data '{
    "jsonrpc": "2.0",
    "method": "platform.importAVAX",
    "params": {
        "username": "YOUR USERNAME",
        "password": "YOUR PASSWORD",
        "to":"YOUR PLATFORM ADDRESS HERE",
        "sourceChain": "X"
    },
    "id": 1
}' -H 'content-type:application/json;' 127.0.0.1:9650/ext/bc/P
```

[Documentation](https://docs.avax.network/v1.0/en/api/platform/#platformimportavax)

You don't need the returned txID.

May take a few minutes for the funds to arrive on the P-Chain.  
Check your AVA balance in the P-Chain account.

```sh
curl -X POST --data '{
  "jsonrpc":"2.0",
  "id"     : 1,
  "method" :"platform.getBalance",
  "params" :{
      "address":"YOUR PLATFORM ADDRESS HERE"
  }
}' -H 'content-type:application/json;' 127.0.0.1:9650/ext/bc/P
```

[Documentation](https://docs.avax.network/v1.0/en/api/platform/#platformgetbalance)

## Become a validator

You need your nodeID

```sh
curl -X POST --data '{
    "jsonrpc":"2.0",
    "id"     :1,
    "method" :"info.getNodeID"
}' -H 'content-type:application/json;' 127.0.0.1:9650/ext/info
```

[Documentation](https://docs.avax.network/v1.0/en/api/info/#infogetnodeid)

Now add your nodeID to the network:

```sh
curl -X POST --data '{
    "jsonrpc": "2.0",
    "method": "platform.addDefaultSubnetValidator",
    "params": {
        "username":"YOUR USERNAME",
        "password":"YOUR PASSWORD",
        "nodeID":"YOUR NODE-ID",
        "rewardAddress":"YOUR PLATFORM ADDRESS HERE",
        "startTime":'$(date --date="15 minutes" +%s)',
        "endTime":'$(date --date="15 days" +%s)',
        "stakeAmount":5000000,
        "delegationFeeRate":0
    },
    "id": 1
}' -H 'content-type:application/json;' 127.0.0.1:9650/ext/P
```

You don't need the returned txID.

Reward address is your P-address to where the money returns after staking incl. rewards.  
Time is in Unix timestamp or in other words seconds sice Jan 1st 1970.  
Here is a [converter](https://www.unixtimestamp.com/).  
You can use a Timestamp for the startTime and endtime directly.  
Now it will calculate start time 15 minutes in the future and end time 15 days in the future. Maximum staking duration is one year minimum one day, minimum will change to 14 days.  
Because we start 15 minutes in the future we will validate for 14 days 23 hours and 45 minutes.

[Documentation](https://docs.avax.network/v1.0/en/api/platform/#platformadddefaultsubnetvalidator)

When you call this:

```sh
curl -X POST --data '{
    "jsonrpc": "2.0",
    "method": "platform.getPendingValidators",
    "params": {},
    "id": 4
}' -H 'content-type:application/json;' 127.0.0.1:9650/ext/P
```

[Documentation](https://docs.avax.network/v1.0/en/api/platform/#platformgetpendingvalidators)

You should find your nodeID and P-address in the list of pending validators. May take a few minutes.

Or a little later this call:

```sh
curl -X POST --data '{
    "jsonrpc": "2.0",
    "method": "platform.getCurrentValidators",
    "params": {},
    "id": 1
}' -H 'content-type:application/json;' 127.0.0.1:9650/ext/P
```

[Documentation](https://docs.avax.network/v1.0/en/api/platform/#platformgetcurrentvalidators)

Gives you the list of current validators. That is the list you want to be in.

You can check the [network explorer](https://explorer.avax.network/) there is a validator searchbar.

Have fun.

## systemd

How to start gecko on every startup and keep it alive in the background.

Inspired by this [article](https://medium.com/@dogusural/how-to-keep-denali-node-alive-on-linux-c16d57e561ca) and some help from here [Running a Go binary as a systemd service](https://fabianlee.org/2017/05/21/golang-running-a-go-binary-as-a-systemd-service-on-ubuntu-16-04/).

Check the article above for info how to monitor your node. I changed the name of the service to `gecko`. So you have to replace "denali" with "gecko" when you use commands from the article above.

We change back to our regular user `ubuntu`. When you are user `gecko` just type `exit` and you should be `ubuntu` again.

```sh
sudo touch /etc/systemd/system/gecko.service
```

as sudo enter the following lines to the file with the editor of your choice, e.g. `sudo nano /etc/systemd/system/gecko.service`

```sh
[Unit]
Description=gecko AVA node
After=network-online.target
Requires=network-online.target
StartLimitIntervalSec=0
[Service]
Type=simple
Restart=always
RestartSec=1
User=user
Group=user
SyslogIdentifier=gecko.service
WorkingDirectory=/opt/user/go/src/github.com/ava-labs/gecko/build
ExecStart=/opt/user/go/src/github.com/ava-labs/gecko/build/avalanche
[Install]
WantedBy=multi-user.target
```

Start service and enable it so it starts after reboot.

```sh
sudo systemctl daemon-reload
sudo systemctl start gecko.service
sudo systemctl enable gecko.service
```

## Backup your stuff

[backup your node](https://medium.com/@otherpaco/backup-your-precious-ava-node-85650b9941a7)

## API calls from another machine

We made all our API calls from the machine gecko is running on. When you want to do it from another machine in your local network you have to make sure your RasPi is getting the same local IP everytime and start gecko with this IP `.ava --http-host=123.my.ip.45`.  
[Documentation](https://docs.avax.network/v1.0/en/references/command-line-interface/)

If you use this method or other arguments and options make sure to add them to `ExecStart` in the systemd `gecko.service`.

```sh
...
SyslogIdentifier=gecko.service
ExecStart=/opt/gecko/go/src/github.com/ava-labs/gecko/build/ava --http-host=123.my.ip.45
[Install]
...
```

You should `sudo systemctl daemon-reload` and restart the `gecko.service`.

It should be possible to do it with your public IP and some port forwarding but I did not try that. I would prefer to connect to my local network via VPN and make the calls "locally".

You should be all set!

You can find me and some other people running a node on a RasPi in the [AVA Discord channels](https://chat.avalabs.org/).
