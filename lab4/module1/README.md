# Module 1

In the first lab module, you will be prepared with the essential materials for development on Ethereum blockchain. You will learn how to set up the **Javascript** development environment on Ubuntu machine. More importantly, you are going to create your first Ethereum account and also send out your first Ethereum transaction!

## Table of Contents

- [Background Knowledge](#background-knowledge)
- [Environment Setup](#environment-setup)
- [Task1: Create an Ethereum account](#task1-create-an-ethereum-account)
- [Task2: MetaMask Installation and Send an Ethereum transaction](#task2-metamask-installation-and-send-an-ethereum-transaction)

## Background Knowledge

_Credit to Steven Cheng and Yu-Tsern Jou for this section of the writeup._

### Smart Contract

In our society, contracts are used to enforce written or spoken agreements. Their effectiveness is supported by the trust of our legal system. However, with the advent of blockchains and cryptocurrencies, people started exploring whether this trust model can be replaced since it is incompatible with the ideas of decentralization and anonymity blockchains proposed. Currently, there is a prevailing solution called smart contracts. Details contracts are programmed and stored on the blockchains instead of being written on papers. In this way, all people can read them before making transactions and the contracts are immutable after they are posted on the blockchains. In this tutorial, we demonstrate the usage of smart contracts, maintaining the validities of Certificate Authorities.

### What is being done now?

Each browser has installed a set of trusted Certificate Authority (CA) certificates. CAs (like Verisign, Symantec, etc.) create root certificates and sign Certificate Signing Requests (CSRs) for other certificates.

An organization that owns a domain that wishes to use SSL/TLS needs a certificate, as per the protocol. However, if the client does not trust that certificate, then he can refuse the connection. A client verifies certificates sent to him by looking at which certificate signed that certificate. If it was signed by a trusted certificate, normally a CA certificate, then the communication continues.

If for some reason, CA certificates are compromised, then the attacker will be able to establish trusted channels of communication between himself and a user. At this point, there must be some kind of update to tell browsers not to trust a certain certificate. But there may be a lag between malicious certificates already in the wild and when a client is notified of a revoked certificate.

### What the blockchain will do?

The blockchain keeps this certificate list live. Essentially, if a CA comes out to say that a certain certificate is being revoked and should no longer be trusted, the CA will simply say so on the blockchain and insert a new trusted certificate (this is assumed to be valid since we are still trusting this CA).

## Environment Setup

### 1. Install Node.js and npm

```bash
$ sudo apt update

$ sudo apt install nodejs

$ sudo apt install npm
```

Verify Node.js and npm has been installed successfully by checking their version: `$ node -v && npm -v`

### 2. Web3.js Library Installation

In this lab, we are going to use the release version _v1.0.0-beta.55_.

`$ npm install web3@v1.0.0-beta.55`

### 3. Infura account signup

In order to connect to the Ethereum network, we need to have access to an Ethereum node. One way to do it could be running our own node with the Ethereum client such as _Geth_, _Parity_, or _Trinity_. However, in such a way requires the client program to have the full replica of Ethereum blockchain data.

A more convenient way to do so would be using the service provided by [Infura](https://infura.io/). Infura provides developers a free way to connect to Ethereum network by accessing the remote Ethereum nodes provided by the Infura team. Therefore, we can utilize this service to connect to the network without running our own local nodes.

- Sign up an account on https://infura.io, and create a project with any name you like.
- After a project has been created, you can view your API key and RPC URL.
  ![Keys](https://i.imgur.com/9d3AV8D.png)

In this lab module, we will use the Ethereum **Ropsten test network** instead of the main network.

Your Ropsten Infura RPC URL should look like this: `ropsten.infura.io/v3/YOUR_ROPSTEN_API_KEY`.

### 4. Connect to Ropsten Network in Node.js console

After getting an Infura account and installing the web3.js library, we can now try to connect to Ethereum Ropsten network. In the terminal, type in the command `$ node` will open up an node.js console. Inside that console, we will instantiate a connection with the Infura RPC URL and make sure the web3.js has been installed successfully.

```bash
$ node

# Inside node.js console
> const Web3 = require("web3")

> const web3 = new Web3("https://ropsten.infura.io/YOUR_API_KEY")

> web3.version
'1.0.0-beta.55'
```

## Task1: Create an Ethereum account

After the dev environment has been configured successfully, we're going to create an Ethereum account so that we can use that account to interact with Ethereum blockchain, e.g. send transactions, or deploy smart contracts. There are multiple ways to do this: either using Ethereum wallet applications such as [MyEtherWallet](https://www.myetherwallet.com/) or [MetaMask](https://metamask.io/), or directly invoke the account creation function through web3.js library. As you might have already guessed it, we're going to write our own javascript code to generate the Ethereum accounts!

First, please make sure that there's no error when creating the web3 instance and could get the correct web3 version.

```javascript
> const Web3 = require("web3")
> const web3 = new Web3("https://ropsten.infura.io/YOUR_API_KEY")
> web3.version
'1.0.0-beta.55'
```

Once confirmed, don't close that node.js console! All the following account creation commands will be executed in the same console. By calling the `web3.eth.accounts.create()` function, it should return an Account object with the address and privateKey attribute.

```javascript
> web3.eth.accounts.create()
Account{
    address: '0xb45d01E85646D3534F8cceA0505128FA2d306d53',
    privateKey: '.......................',
    .
    .
    .
}
```

After you get that Account object, the address will be the unique public identifier and the privateKey will be the credential for accessing that account. Therefore, the private key should definitely be kept secretly.

Here we are going to store that private key as the environment variable and load it only when we need to access that private key. Open up another terminal to use the following command to store it in environment

```bash
$ export ETH_PRIVATE_KEY="YOUR_PRIVATE_KEY"

# verify
$ echo $privateKey
```

However if we're doing in this way, every time we start another new shell session, we will need to re-export those secret values. Hence, a better way is to modify your `bashrc` file (if you're using bash) and put that export statement inside that bashrc file. For more information, please reference this tutorial: https://www.digitalocean.com/community/tutorials/how-to-read-and-set-environmental-and-shell-variables-on-a-linux-vps

```bash
# open up bashrc file, use whatever text editor you like!
$ vim ~/.bashrc

---
# put the export statement at the end of your bashrc file
export ETH_PRIVATE_KEY="YOUR_PRIVATE_KEY"
---
# Force the current bash session to read the bashrc file
$ source ~/.bashrc
```

Once the private key has been stored as the environment variable, let's jump back to the node.js console and try to read it from the environment using the javascript code.

```javascript
// The address could be hardcoded, since that is supposed to be publicly known.
> const address = "0x......."

> const privateKey = Buffer.from(process.env.ETH_PRIVATE_KEY, 'hex')
> console.log(privateKey)
```

## Task2: MetaMask Installation and Send an Ethereum transaction

First, go to https://metamask.io/ to install MetaMask, which is a browser extension for accessing Ethereum network. Think of this as a GUI for Ethereum wallet application, so that you can view (your) account balance, view (your) transaction history, and send new transactions by using this application instead of writing your own code for achieving the same tasks. To use this application, you can either choose to create another account with the Metamask UI, or import existing accounts by providing a private key string or JSON key file. In this lab, we're going to _import the Ethereum account you created in Task1_!

Therefore, select "Import Account", and paste the private key that you generated in Task 1.

<img src="https://i.imgur.com/hPWy7Px.png" alt="drawing" width="300"/>

Noted that you may need to have to create a dummy account first in order to access the interface shown above. In this case, you can just creat a new account with Metamask and then import the Ethereum account we have created.

Once you've imported your account into MetaMask, you should able to view your account balance which should be 0 by default and switch to different network environments by clicking the drop-down menu on the top right corner. In this module, all the transactions that we'll do will be on the Ropsten test network, so select "Ropsten Test Network" and don't change it until you finish all the tasks!

<img src="https://i.imgur.com/b3Beahz.png" alt="drawing" width="300"/>

Since this lab module is based on the **Ethereum test network (Ropsten)**, this means that all the Ethers used in this network do not have real monetary value. In other words, the ETHs exist on all test networks are worthless and we could get them for free!

Go to https://faucet.metamask.io/ and enter your account address. This will initiate a transaction that gives you 1 ETH on the Ropsten test network.
![faucet2](https://i.imgur.com/fUd4JLL.png)

Now, let's try to send out our first Ethereum transaction! Ask one of your friends to give you his/her account address, and try to send him/her some ETH! In MetaMask, click the "Transfer" button, then paste your friend's account address, then click "Confirm" to submit the transaction. That's it! Wait a few seconds till that transaction has been _confirmed_, and ask your friend if he/she has received the ETH sent from you.

<img src="https://i.imgur.com/BQFh8j4.png" alt="drawing" width="300"/>

<img src="https://i.imgur.com/j011bdH.png" alt="drawing" width="300"/>

<img src="https://i.imgur.com/rWbZoD9.png" width="300"/>

Click on the transaction you just sent in MetaMask, and you should see more information about that transaction. Click the "View on Etherscan" button on the top right section, and you should be directed to the Etherscan web page which presents more comprehensive information about that transaction.
![etherscan](https://i.imgur.com/vpKsL0u.png)

After learning how to create Ethereum accounts and send transactions using a graphical interface, now you should have a better understanding of Ethereum and using web3.js to interact with the Ethereum network. In the next lab module, you will learn how to write your first smart contract, deploy it and read the content of smart contracts all using web3.js library. Please refer to [this great tutorial](https://www.dappuniversity.com/articles/web3-js-intro) by Gregory if you're willing to learn more about web3.js and Ethereum development.

## References

- https://github.com/ethereum/web3.js/
- https://web3js.readthedocs.io/en/v1.2.1/
- https://www.dappuniversity.com/articles/web3-js-intro
