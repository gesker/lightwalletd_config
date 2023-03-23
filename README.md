# Setup Light Wallet Daemon (Zcash Blockchain Node) on a Development Workstation


This document is a guide for installing lightwalletd, on BOTH **mainnet** and **testnet**, on a development workstation. Further, lightwalletd will be configured to run again the orginal zcash node (zcashd) AND run against a newer implentation of the zcash node called zebra (zebrad) which is still under development.

Lightwalletd provides an interface for mobile zcash clients.


**As of the creation date of this document a Debian package does not appear to be avilable for lightwalletd in the (zcash apt repository)[https://apt.z.cash/].** 



## Preamble


This guide has been generated using notes I have collected from various sources. Some of these notes relate to system administration tasks on Debian Linux and how they relate to running the Light Wallet Daemon (lightwalletd) on Debian. 

The single best information resource regarding zebra can be found in its git repository; (lightwalletd)[https://github.com/zcash/lightwalletd].  

The single best information resource for (Debian)[https://www.debian.org] is, of course, the (Debian Home Page)[https://www.debian.org] more specifically the (Debian Users' Manuals)[https://www.debian.org/doc/user-manuals]. (The Debian Administrator's Handbook)[https://debian-handbook.info/] is also an excellent resource. 

This lightwalletd setup "recipe" assumes that your Debian development workstation already has a copy of the **ORIGINAL** (zcashd)[https://github.com/zcash/zcash] full node software installed and configured according to the instructions published in my (github repository here)[https://github.com/gesker/zcashd_config]. 


This lightwalletd setup "recipe" assumes that your Debian development workstation already has a copy of the **NEW** (zebrad)[https://github.com/ZcashFoundation/zebra] full node software installed and configured according to the instruction published in my (github repository here)[https://github.com/gesker/zebrad_config]. 


Completing the instructions found in those instructions is NOT NECESSIARILY a prerequisite to completing this recipe; just be aware of the ports assigned to zcashd and zebrad. This document will also follow the same pattern found in the earlier guides of installing the software and configuring a Linux Service so that the node software is controlled by the systemd software provided by Debian. The zebra node operate on BOTH the zcash *mainnet* and *testnet* blockchains. Because, upon completion of BOTH previous sets of instructions, your workstation will have *zcashd* running on ports *8233* (mainnet) and *18233* (testnet) and zebrad running on the NON-standard ports of *8243* (for mainnet) and *18243* (for testnet).


If you were to complete ALL THREE sets of instrutions your configuration will resemble:
```
[zcash blockchain network] <--> 8233:[zcash mainnet]:8232 <--> [lightwalletd]:9067(Rest)&9068(gRpc) <--> wallet software
[zcash blockchain network] <--> 18233:[zcash testnet]:18232 <--> [lightwalletd]:9069(Rest)&9070(gRpc) <--> wallet software
[zcash blockchain network] <--> 8234:[zebra mainnet]:8242 <--> [lightwalletd]:9071(Rest)&9072(gRpc) <--> wallet software
[zcash blockchain network] <--> 18234:[zebra testnet]:18242 <--> [lightwalletd]:9073(Rest)&9074(gRpc) <--> wallet software
```

So, you will have... 

TWO instances of zcash; mainnet and testnet.
TWO instances of zebra; mainnet and testnet.
FOUR instances of lightwalletd; communicating with BOTH zcashd and zebrad on BOTH mainnet and testnet.

Upon completion of these guides you will have just about ALL the software you will need to explore and develop applications against the zcash blockchain.


## License


    Copyright (c) 2023 Dennis R. Gesker.
    Permission is granted to copy, distribute and/or modify this documenet spe
    under the terms of the GNU Free Documentation License, Version 1.3
    or any later version published by the Free Software Foundation;
    with no Invariant Sections, no Front-Cover Texts, and no Back-Cover Texts.
    A copy of the license is included in the section entitled "GNU

## Goals


- Install Light Wallet Daemon (lightwalletd)[https://github.com/zcash/lightwalletd]
- Configure lightwalletd to automatically start using systemd
  - Mainnet (mainnet) communicating with **zcashd** on localhost
  - Testnet (testnet) communicating with **zcashd** on localhost
  - Mainnet (mainnet) communicating with **zebrad** on localhost
  - Testnet (testnet) communicating with **zebrad** on localhost
- Configure systemd to control lightwalletd


## Non-Goals


- Guarantee Security of the configuration
- This is for a personal workstation only!


## Miscellaneous 


- I am not part of the (lightwalletd)[https://github.com/zcash/lightwalletd] project
- I am not part of the (zcash)[https://github.com/zcash/zcash] project
- I am not part of the (zebra)[https://github.com/ZcashFoundation/zebra] project
- I DO use this configuration on my own workstation
  - Use at your own risk!
- Pull requests to improve these notes are welcome


## Requirements


- Debian 12 (Bookworm)
- Ability to sudo on the system
- Ability to use a text editor and navigate directories
- About 400 GB of disk space in your home *~/* directory


## Step 1 - Install some prerequisites

```
sudo apt update
sudo apt install wget git 
```


## Step 2 - Install Go

We will need the go development tools. Download and install rust. 

```
wget https://go.dev/dl/go1.20.2.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.20.2.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
go version
```

**Tip:** If you anticipate working through these instruction more than once consider adding `export PATH=$PATH:/usr/local/go/bin` to the end of your ~/.profile so that the go tools are available on your next login.


## Step 2 - Download & Compile Source Code

As of the creation of this document lightwalletd is at *release v0.4.13* (v0.4.13)[https://github.com/zcash/lightwalletd/releases/tag/v0.4.13] and as far as I have been able to determine a Debian apt package is not yet available. We will install the software from the source code and use the main branch instead of the release branch.


```bash
mkdir ~/Development/
cd ~/Development/
git clone git@github.com:zcash/lightwalletd.git;
cd ~/Development/lightwalletd
make
```


The build process will complete and the new lightwalletd binary will be created at `~/Development/lightwalletd/lightwalletd`.



## Step 3  - Create Storage Location for Zcash Blockchain



The zcash blockchain is pretty large. Let's store these files under the ~/zebrad directory.

*Tip:* If you have a LARGER physical drive mounted on the system create a symbolic link from ~/zebrad to that larger drive. 


```bash
mkdir ~/lightwalletd;
mkdir ~/lightwalletd/mainnet_zcashd/;
mkdir ~/lightwalletd/testnet_zcashd/;
mkdir ~/lightwalletd/mainnet_zebrad/;
mkdir ~/lightwalletd/testnet_zebrad/;
```


## Step 5 - Copy and UPDATE the configuration files


Create a new sub-directory and copy the configuraiton files to this new folder.

```bash
wget https://github.com/gesker/lightwalletd_config/lightwalletd_mainnet_zcashd.yml -O ~/lightwalletd/lightwalletd_mainnet_zcashd.yml
wget https://github.com/gesker/lightwalletd_config/lightwalletd_testnet_zcashd.yml -O ~/lightwalletd/lightwalletd_testnet_zcashd.yml
wget https://github.com/gesker/lightwalletd_config/lightwalletd_mainnet_zebrad.yml -O ~/lightwalletd/lightwalletd_mainnet_zebrad.yml
wget https://github.com/gesker/lightwalletd_config/lightwalletd_testnet_zebrad.yml -O ~/lightwalletd/lightwalletd_testnet_zebrad.yml
wget https://github.com/gesker/lightwalletd_config/zebrad.conf.mainnet -O ~/lightwalletd/zebrad.conf.mainnet
wget https://github.com/gesker/lightwalletd_config/zebrad.conf.testnet -O ~/lightwalletd/zebrad.conf.testnet
```


**Important:** In BOTH of the files above CHANGE *yourUserName* to your actual username on the system.

**Note:** The yml files for zcashd will use the [~/.zcash/zcash.conf.mainnet](https://github.com/gesker/zcashd_config/blob/main/zcash.conf.mainnet) and [~/.zcash/zcash.conf.testnet](https://github.com/gesker/zcashd_config/blob/main/zcash.conf.testnet) files created in the earlier guide **BUT** we did not create these conf file equivilents for zebrad and lightwalletd must have a conf file for zebrad so we will have minimal conf files for zebrad.




## Step 6 - Copy and UPDATE the systemd files


Copy down the configuration files for systemd, enable the services, and start the services


```bash
sudo wget https://github.com/gesker/lightwalletd_config/lightwalletd_mainnet_zcashd.service -O ~/etc/systemd/system/lightwalletd_mainnet_zcashd.service
sudo wget https://github.com/gesker/lightwalletd_config/lightwalletd_testnet_zcashd.service -O ~/etc/systemd/system/lightwalletd_testnet_zcashd.sevice
sudo wget https://github.com/gesker/lightwalletd_config/lightwalletd_mainnet_zebrad.service -O ~/etc/systemd/system/lightwalletd_mainnet_zebrad.service
sudo wget https://github.com/gesker/lightwalletd_config/lightwalletd_testnet_zebrad.service -O ~/etc/systemd/system/lightwalletd_testnet_zebrad.service
```


**Important:** In BOTH of the files above CHANGE *yourUserName* to your actual username on the system.


Reload the systemd daemon so that the new files are recognized and start lightwalletd on BOTH mainnet and testnet on BOTH zcashd and zebrad.

```bash
sudo systemctl daemon-reload 
sudo systemctl start lightwalletd_mainnet_zcashd;
sudo systemctl start lightwalletd_testnet_zcashd;
sudo systemctl start lightwalletd_mainnet_zebrad;
sudo systemctl start lightwalletd_testnet_zebrad;
```

*Discussion:* For the most part you will only need one setup at a time. Most likely lightwalletd (testnet_zcashd) <-> zcashd (testnet) <-> zcash blockchain. But, now you have the flexibility to test against all of these scenarios. This configuration will most likely be suited for your automation pipeline. Typically, I leave zcashd and zebrad running on both testnet and mainnet because the time limiting factor is the blockchain syncronization. I typically only have a single instance of lightwalletd running. Just stop the instance of lightwalletd you don not need.


Tail the log files to confirm that the services are running

```bash
tail -f ~/lightwalletd/mainnet_zcashd/lightwalletd.log  # Ctrl-C to exit tail
tail -f ~/lightwalletd/testnet_zcashd/lightwalletd.log  # Ctrl-C to exit tail
tail -f ~/lightwalletd/mainnet_zebrad/lightwalletd.log  # Ctrl-C to exit tail
tail -f ~/lightwalletd/testnet_zebrad/lightwalletd.log  # Ctrl-C to exit tail
```

Instruct systemd to start lightwalletd when when your workstation is restarted.
```bash
sudo systemctl enable lightwalletd_mainnet_zcashd; # Optional
sudo systemctl enable lightwalletd_testnet_zcashd; # Optional
sudo systemctl enable lightwalletd_mainnet_zebrad; # Optional
sudo systemctl enable lightwalletd_testnet_zebrad; # Optional
```

It will take a short amount for lightwalletd to fully syncronize - assuming zcashd and/or zebrad is already fully syncronized! in the lightwalletd.log files you will see entries similar to...



```
{"app":"lightwalletd","level":"info","msg":"Adding block to cache 442234 00000000000d5d5b4179c562a6eb01d7e6fa11fa0589befa1257ca6ed15b037e","time":"2023-03-07T11:27:18-07:00"}
{"app":"lightwalletd","level":"info","msg":"Adding block to cache 442576 00000000029e5f9e03367080bc78fb27657b16fc0e0ceb7d694b06695a6be6a1","time":"2023-03-07T11:27:22-07:00"}
{"app":"lightwalletd","level":"info","msg":"Adding block to cache 442925 00000000001b36b20d7ddb52743e9e452c01c7e59dd2ac0d7089b0913797fd0a","time":"2023-03-07T11:27:26-07:00"}
{"app":"lightwalletd","level":"info","msg":"Adding block to cache 443387 0000000000a5a7bc68af029a99aea0e04c14aa37a910b3c7d2cd71ec8f91f565","time":"2023-03-07T11:27:30-07:00"}
{"app":"lightwalletd","level":"info","msg":"Adding block to cache 443911 00000000020037cf2e84ce93c0f7e4ef61fb826d6abda699e53c0850921783dc","time":"2023-03-07T11:27:34-07:00"}
```


## Additional Information Sources


- [Zcash Documentation](https://zcash.readthedocs.io/en/latest/)
- [Electric Coin Company](https://electriccoin.co/)
- [Zcash Forums](https://forum.zcashcommunity.com/)
- [Debian Adminsitrator's Handbook](https://debian-handbook.info/)

## This Document

- Feedback welcome
- Pull requests welcome
- Original Located at [Github](https://github.com/gesker/lightwalletd_config)



