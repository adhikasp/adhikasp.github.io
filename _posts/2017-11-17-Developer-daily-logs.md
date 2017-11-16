---
title: "Developer daily log (hopefully) - 2017-11-17"
layout: single
excerpt: "Trying to dive into Ethereum. Use testnet. Send some ether. Interact with smart contract."
sitemap: true
author_profile: false
category: "developer"
tags:
  - "English"
  - "Daily dev log"
---

Trying to dive into Ethereum

## Getting some ether

Reading about rinkeby. Apparently this could be used for play and testing in real network. 
Unfortunately I already ~~waste~~ invest 50k rupiah to buy real ether in bitcoin.co.id. Meh.

Use metamask. Generate some address. Go to <https://www.rinkeby.io/#faucet> to get some free ether.

Send some ether to my friends address for fun XD
<https://rinkeby.etherscan.io/address/0x475b48dd60a57e9da28b282b3e5d5c8d181980e1>

## Rinkeby geth 

Trying to connect to rinkeby network with geth.

geth --networkid=4 --datadir="D:\Program\Rinkeby Blockchain" --syncmode=light --ethstats='janjim2:helloworld@stats.rinkeby.io' --bootnodes=enode://a24ac7c5484ef4ed0c5eb2d36620ba4e4aa13b8c84684e1b4aab0cebea2ae45cb4d375b77eab56516d34bfbd3c1a833fc51296ff084b770b94fb9028c4d25ccf@52.169.42.101:30303?discport=30304

Start time 16 Nov 2017 11:17 GMT

Always got `WARN [11-16|23:18:04] Stats login failed	err=unauthorized`
Maybe need to change the `--ethstats` value?
https://ethereum.stackexchange.com/a/29650 Need to change to unique
node name, but doesn't work too :(

Pause at 17 Nov 2017 00:16 GMT. Syncing 434048 from 1.5M blocks.
It just takes too long and I don't want to waste electricity and
bandwith.
I doesn't really need it to learn anyway. I can use [online viper
compiler](https://viper.tools/) and deploy/interact with smart contract using [MyEtherWallet](https://myetherwallet.com).


## Creating ENS

Try to register a fancy address name, adhikasp.test.

ENS public resolver https://rinkeby.etherscan.io/address/0xb14fdee4391732ea9d2267054ead2084684c0ad8#code

Can't make it works.


## Deploying Hello World contract

Ty for the tutorial https://www.reggie.io/blog/deploying-ethereum-viper/

deployed on rinkenby https://rinkeby.etherscan.io/address/0xc7ed1f68bf9cd3b67500e7b6b9549352bd8cf452

Reading and modifying data works. But I can't withdraw the contract's
earnings using the owner addres. TX always failing. HALP!

