# Module 2

In this lab module, we are going to implement a minimal _decentralized_ PKI (Public Key Infrastructure) deployed on Ethereum Ropsten network. This lab module has been fully tested on nodes provisioned by [GENI](https://www.geni.net/) with Ubuntu version 18.04.1 LTS. After completing this module, you will learn how to write Ethereum smart contract in Solidity, a programming language, to read data from Ethereum blockchain, write transactions, and deploy smart contracts using web3.js library, so to set up HTTPS connection on Nginx server. The goal is eventually for you to have a solid understanding of PKI and X.509 digital certificates, supported by blockchain.

## Table of Contents

- [GENI Topology](#geni-topology)
- [Task0: Environment Setup](#task0-environment-setup)
- [Task1: RootCA - generate self-signed root certificate](#task1-rootca---generate-self-signed-root-certificate)
- [Task2: RootCA - Deploy Smart Contract](#task2-rootca---deploy-smart-contract)
- [Task3: Server - Generate CSR(Certificate Signing Request)](#task3-server---generate-csrcertificate-signing-request)
- [Task4: RootCA - Sign CSR and Generate Certificate](#task4-rootca---sign-csr-and-generate-certificate)
- [Task5: Server - Retrieve Certificates](#task5-server---fetch-certificates)
- [Task6: Server - Nginx Server Setup](#task6-server---nginx-server-setup)
- [Task7: Client - Test the HTTPS Connection](#task7-client---test-the-https-connection)

## GENI Topology

- RootCA
- Server
- Client

![topology](https://i.imgur.com/gdrjsDI.png)

In this topology, the `RootCA` node serves as a Certificate Authority (CA). But instead of maintaining the certificates locally, we will create an Ethereum smart contract in a decentralized medium to store a certificate, and allow different parties to interact with each other in a "decentralized way". So the `RootCA` node will be responsible for creating the smart contract, signing the CSR (Certificate Signing Request), and publishing the signed certificates. The main task for the `Server` node is hosting an Nginx web server with SSL/TLS enabled and running. To support this, the `Server` node will need to create a CSR, ask the `RootCA` node to sign that, and retrieve the signed certificate from the Ethereum blockchain. Lastly, the `Client` node will test the HTTPS connection with the Nginx server on `Server` node.

## Task0: Environment Setup

Please refer to the `Environment Setup` section in "Module 1" for specific software or library installation.

Here's a list of software/library packages that you need to install on each node:

RootCA

- Node.js & npm
- web3.js library (_v1.0.0-beta.55_)
- ethereumjs-tx library (_v1.3.7_, by running `$ npm install ethereumjs-tx@1.3.7` in the terminal)

Server

- Node.js & npm
- web3.js library (_v1.0.0-beta.55_)
- ethereumjs-tx library (_v1.3.7_, by running `$ npm install ethereumjs-tx@1.3.7` in the terminal)

Client

- Node.js & npm
- web3.js library (_v1.0.0-beta.55_)

## Task1: RootCA - Generate self-signed root certificate

SSH into the RootCA node, and run the following command to generate a private key and a self-signed root certificate signed by that private key.

```bash
$ openssl req -x509 -newkey rsa:4096 -keyout rootca_key.pem -out root_cert.crt -days 365

# The following input prompt should show up after entering the passphrase for your private key.
Country Name (2 letter code) [AU]:US
State or Province Name (full name) [Some-State]:Maryland
Locality Name (eg, city) []:Baltimore
Organization Name (eg, company) [Internet Widgits Pty Ltd]:JHU
Organizational Unit Name (eg, section) []:ISI
Common Name (e.g. server FQDN or YOUR name) []:jhuisi-ethcert-root.com
Email Address []:
```

## Task2: RootCA - Deploy Smart Contract

We are going to use [Solidity](https://en.wikipedia.org/wiki/Solidity) version 0.4.25 to write smart contracts on Ethereum blockchain, and we will use [Remix](https://remix.ethereum.org/#optimize=false&evmVersion=null&version=soljson-v0.5.11+commit.c082d0b4.js&appVersion=0.7.7), which is an IDE for Ethereum developers in Solidity, to compile the code, test/debug the contract code and deploy the smart contracts in various ways.

> Solidity is a statically typed, contract-oriented, high-level language for implementing smart contracts on the Ethereum platform. (https://github.com/ethereum/solidity)

_NOTE: In this doc, we will demonstrate the deployment process using the old version of Remix, which has a different user interface. You can choose to use either the newer Remix version or the older one._

The following shows a smart contract that we will use for this lab module:

```javascript
pragma solidity ^0.4.25;

contract cert {
    string public rootCert;
    string public csrFile;
    string public serverCert;
    bool public isValid = false;

    constructor(string file) public {
        rootCert = file;
    }

    function uploadCSR(string file) public returns (bool) {
        csrFile = file;
        return true;
    }

    function uploadCert(string file) public returns (bool) {
        serverCert = file;
        isValid = true;
        return true;
    }

    function checkCertStatus() public view returns (string) {
        if (isValid == true) {
            return "Certificate is valid!";
        } else {
            return "Certificate is invalid!";
        }
    }

    function revokeCertificate() public returns (bool) {
        isValid = false;
        serverCert = "XXX";

        return true;
    }
}
```

Once the Remix IDE is opened up in the browser (We recommend to use Google Chrome browser for Remix), create a new file with **.sol** file extension, and put the smart contract above into the code editor. You would probably notice that there are a lot of syntax errors showing up.

![remix errors](https://i.imgur.com/BdRm3w2.png)

The reason that those syntax errors pop up is due to the fact that we are using the Solidity version 0.4.25, but Remix IDE by default loads the compiler with Soldiity version 0.5.1, which is not backward compatible with the codes written in previous versions. Developers could select the Solidity version they're going to use with the code `pragma solidity ^SOLIDITY_VERSION;`.

To solve the compilation errors, all you need to do is to select the right compiler version! Click the "Select new compiler version" button, and select the version **0.4.25+commit.59dbf8f1**. Once the compiler version has been chosen correctly, Remix should automatically re-compile the code. At this time, our smart contract should be successfully compiled without any fatal errors.

![remix correct compiler version](https://i.imgur.com/c0DHXsL.png)

Next, we are going to deploy this smart contract to the Ethereum Ropsten test network. Before proceeding further, please make sure you have imported an Ethereum account into MetaMask and there are some ETHs on Ropsten Network. Once confirmed, let the MetaMask stay on Ropsten Network.

Now, in Remix's right section, switch to the "Run" tab. In the Run tab, switch the Environment from "Javascript VM" to **"Injected Web3"**. Once switched, your Ethereum account address and the balance on Ropsten network should be loaded. If you got the outcome similar to the screenshot below, then you are good to go to deploy your first Ethereum smart contract!

<img src="https://i.imgur.com/KdXwJXj.png" alt="drawing" width="300"/>

To deploy our smart contract, we also need to initialize the rootCert variable, as specified in the _constructor_ function of it. So, go back to the `RootCA` node and copy the self-signed root certificate that you created in Task1. One simple way to do that is to use the command line tool `cat` to print out the file, then copy the printed outcome. The certificate file should start with the indicating string `-----BEGIN CERTIFICATE-----`.

Now go back to Remix, paste the certificate into the field next to the "Deploy" button. Don't forget to put the **double quote** before and after the certificate file, as that will be stored as a string variable.

<img src="https://i.imgur.com/zPqU9dP.png" alt="drawing" width="300"/>

After clicking the "Deploy" button, a MetaMask window will pop up to ask for confirmation. Click the "Confirm" button, wait for a few seconds, and your smart contract should de deployed!

<img src="https://i.imgur.com/XJNf9pc.png" alt="drawing" width="300"/>

You could verify the status of that contract deployment transaction by viewing it at MetaMask, or go to the EtherScan for more detailed information. In MetaMask, when you click on the transactions, there should be a "View on EtherScan" button to directly guide you to the EtherScan for that transaction.

<img src="https://i.imgur.com/u99e4u9.png" alt="drawing" width="300"/>

Before proceeding to the next task, please make sure that your smart contract is deployed successfully!

![etherscan](https://i.imgur.com/XGOqPtu.png)

## Task3: Server - Generate CSR(Certificate Signing Request)

First, let's generate a new Certificate Signing Request (CSR) file using openssl toolkit.

```bash
# Generate new private key
$ sudo openssl genrsa -out jhu-server.key 2048

# Create CSR with that private key
$ sudo openssl req -new -key jhu-server.key -out jhu-server.csr

Country Name (2 letter code) [AU]:US
State or Province Name (full name) [Some-State]:Maryland
Locality Name (eg, city) []:Baltimore
Organization Name (eg, company) [Internet Widgits Pty Ltd]:JHU
Organizational Unit Name (eg, section) []:ISI
Common Name (e.g. server FQDN or YOUR name) []:jhuisi-ethcert-server.com
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

Once CSR has been generated successfully, we need to upload that CSR file to Ethereum Smart Contract so that the RootCA node can sign the CSR and generate the corresponding certificate. Before that, let's try to read the root certificate uploaded by RootCA node in step 2.

To instantiate a smart contract object in node.js console using web3.js library, two pieces of information are necessary: the **address of the smart contract** and the **smart contract ABI** (Application Binary Interface). For more information about ABI, please refer to the [Wiki page](https://en.wikipedia.org/wiki/Application_binary_interface) and some [stackexchange posts](https://ethereum.stackexchange.com/questions/234/what-is-an-abi-and-why-is-it-needed-to-interact-with-contracts).

You are supposed to get the address of the smart contract in task2 when deploying your first smart contract. The address should be shown on the Etherscan webpage when viewing your contract deployment transaction. Regarding the contract ABI, please go to the Remix IDE, under the "Compile" Tab in the right section, you should find an icon named "ABI" to copy the content of the ABI. The ABI that you got should be similar to the example provided in the following code snippet, except that we had beautified my ABI for better formatting.

<img src="https://i.imgur.com/uVrdEiG.png" alt="drawing" width="300"/>

After having these two pieces of information, we could try to instantiate a contract object, and make a function call to read the root certificate value stored in the Ethereum smart contract.

```javascript
// Read the smart contract!
> const Web3 = require("web3")

> const web3 = new Web3("https://ropsten.infura.io/YOUR_API_KEY")

> web3.version

// You should find the address of your smart contract when you deployed it at task2.
> const contractAddress = "YOUR_SMART_CONTRACT_ADDRESS"

> const contractABI = YOUR_SMART_CONTRACT_ABI

// ex: const contractABI = [{"constant":false,"inputs":[],"name":"revokeCertificate","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"rootCert","outputs":[{"name":"","type":"string"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[],"name":"serverCert","outputs":[{"name":"","type":"string"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[],"name":"isValid","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[],"name":"csrFile","outputs":[{"name":"","type":"string"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"file","type":"string"}],"name":"uploadCSR","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"name":"file","type":"string"}],"name":"uploadCert","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"checkCertStatus","outputs":[{"name":"","type":"string"}],"payable":false,"stateMutability":"view","type":"function"},{"inputs":[{"name":"file","type":"string"}],"payable":false,"stateMutability":"nonpayable","type":"constructor"}]

> const contract = new web3.eth.Contract(contractABI, contractAddress)

> contract.methods.rootCert().call().then((cert) => {console.log(cert)});
```

The last javascript command should give you output like this, which prints out the root certificate to the console:

<img src="https://i.imgur.com/ld3uCiM.png" alt="drawing" width="500"/>

Next, we will upload the CSR file that we generate earlier, which will be a **write transaction** to Ethereum smart contract, modifying the state of Ethereum network. To do so, we will need to construct a transaction object, sign it with the private key, and send out the transaction. We will utilize the library _ethereumjs-tx_ to accomplish the above-mentioned tasks.

Install the version 1.3.7 library: `$ npm install ethereumjs-tx@1.3.7`

```javascript
> const Tx = require("ethereumjs-tx")

> const account1 = 'YOUR_ETH_ACCOUNT_ADDR'
// ex: const account1 = '0x6De84c79602B544Bed2a8e1611b830B93c084784'

> const privateKey1 = Buffer.from(process.env.PRIVATE_KEY_1, 'hex')

> const fs = require('fs');

> var csrFile;

> fs.readFile('jhu-server.csr', 'utf-8' , (err, data) => {
	if (err) throw err;
	csrFile = data;
  })

> web3.eth.getTransactionCount(account1, (err, txCount) => {

    const txObject = {
        nonce: web3.utils.toHex(txCount),
        gasLimit: web3.utils.toHex(800000),
        gasPrice: web3.utils.toHex(web3.utils.toWei('10', 'gwei')),
        to: contractAddress,
        data: contract.methods.uploadCSR(csrFile).encodeABI()
    }

    const tx = new Tx(txObject)
    tx.sign(privateKey1)

    const serializedTx = tx.serialize()
    const raw = '0x' + serializedTx.toString('hex')

    web3.eth.sendSignedTransaction(raw, (err, txHash) => {
				if (err) throw err;
        console.log('txHash:', txHash)
    })
})

txHash: 0x0f4131737c04493cba55702ccc911c8f40bba5f972dc0435e325c03529933c34
```

After successfully get the transaction hash, try to read the csrFile variable from the Ethereum smart contract

```javascript
> contract.methods.csrFile().call().then((result) => {console.log(result)})
```

<img src="https://i.imgur.com/fpxOEMl.png" alt="drawing" width="500"/>

## Task4: RootCA - Sign CSR and Generate Certificate

```javascript
> const Web3 = require("web3")
> const web3 = new Web3("https://ropsten.infura.io/YOUR_API_KEY")
> const contractAddress = "YOUR_CONTRACT_ADDRESS"
> const contractABI = YOUR_CONTRACT_ABI
> const contract = new web3.eth.Contract(contractABI, contractAddress)

> const fs = require("fs")

> var csrFile

> contract.methods.csrFile().call().then((result) => {csrFile = result})

> fs.writeFile("csrFromServer.csr", csrFile, (err) => {
		if (err) throw err;
		console.log("csr file has been saved!");
})
```

Now on the RootCA node, a CSR file named `csrFromServer.csr` should have been saved locally.

```bash
# Sign the CSR
$ openssl x509 -req -days 365 -in csrFromServer.csr -CA root_cert.crt -CAkey rootca_key.pem -CAcreateserial -out server.crt

# Verify the output server has the correct subject and issuer
$ openssl x509 -in server.crt -text -noout
```

The next step is to upload that signed certificate to the smart contract so that the server node can utilize that signed certificate to setup a secure https connection. Don't forget to install ethereumjs-tx library! `$ npm install ethereumjs-tx@1.3.7`

```javascript
> const Tx = require("ethereumjs-tx")

> const fs = require("fs")

> const account1 = 'YOUR_ETH_ACCOUNT_ADDR'
// ex: const account1 = '0x6De84c79602B544Bed2a8e1611b830B93c084784'

> const privateKey1 = Buffer.from(process.env.PRIVATE_KEY_1, 'hex')

> var serverCert;

> fs.readFile('server.crt', 'utf-8', (err, data) => {
		if (err) throw err;
		serverCert = data;
})

> web3.eth.getTransactionCount(account1, (err, txCount) => {

    const txObject = {
        nonce: web3.utils.toHex(txCount),
        gasLimit: web3.utils.toHex(1500000),
        gasPrice: web3.utils.toHex(web3.utils.toWei('10', 'gwei')),
        to: contractAddress,
        data: contract.methods.uploadCert(serverCert).encodeABI()
    }

    const tx = new Tx(txObject)
    tx.sign(privateKey1)

    const serializedTx = tx.serialize()
    const raw = '0x' + serializedTx.toString('hex')

    web3.eth.sendSignedTransaction(raw, (err, txHash) => {
				if (err) throw err;
        console.log('txHash:', txHash)
    })
})
txHash: 0x195a2d6b0aa9a54437bb598afa9f6884f18ea8c48e817d1e2a2c8784cfd0e843

// Make sure that the data has successfully been written to the smart contract
> contract.methods.serverCert().call().then((cert) => {console.log(cert)})
```

<img src="https://i.imgur.com/LZqJ70Q.png" alt="drawing" width="500"/>

## Task5: Server - Fetch Certificates

In the server's node.js console, read the signed server certificate and root certificate and store them locally.

```javascript
> const fs = require("fs")
> var serverCert;
> contract.methods.serverCert().call().then((cert) => {serverCert = cert})
> fs.writeFile("server.crt", serverCert, (err) => {
		if (err) throw err;
		console.log("certificate file has been saved.")
})
> var rootCert;
> contract.methods.rootCert().call().then((cert) => {rootCert = cert})
> fs.writeFile("root_cert.crt", rootCert, (err) => {
		if (err) throw err;
		console.log("certificate file has been saved.")
})
```

Verify the certificate: `$ openssl x509 -in server.crt -text -noout`

## Task6: Server - Nginx Server Setup

First, install Nginx: `$ sudo apt install nginx`

Verify that Nginx server is running:

```bash
$ systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: e
   Active: active (running) since Sat 2019-07-13 16:05:06 EDT; 3min 37s ago
     Docs: man:nginx(8)
  Process: 31638 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (co
  Process: 31637 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_proces
 Main PID: 31639 (nginx)
    Tasks: 2 (limit: 1109)
   CGroup: /system.slice/nginx.service
           ├─31639 nginx: master process /usr/sbin/nginx -g daemon on; master_p
           └─31640 nginx: worker process
```

Now get the IP address of the server with the `$ hostname -i` command. Then try to curl the Nginx server both from the server node and other nodes. The IP address in your case might be different. For example, the command we use is similar to: `$ curl http://172.17.1.1`.

<img src="https://i.imgur.com/1g9omEr.png" alt="drawing" width="500"/>

Before modifying the Nginx configuration file to enable SSL connections, first we need to concatenate the root certificate file and server certificate file:
`$ cat server.crt root_cert.crt > cert_bundled.crt`

Then let's move the bundled certificate file and server's private key to /etc/ssl directory:

`$ sudo mv cert_bundled.crt /etc/ssl/`

`$ sudo mv jhu-server.key /etc/ssl/`

Now open up the file of `/etc/nginx/sites-enables/default` with your favorite text editor. Add the following three lines to enable SSL connections:

```
listen 443 ssl;
ssl_certificate /etc/ssl/cert_bundled.crt;
ssl_certificate_key /etc/ssl/jhu-server.key;
```

<img src="https://i.imgur.com/9LUMf7S.png" alt="drawing" width="500"/>

Finally, restart the Nginx server: `$ sudo nginx -s reload`. If everything has been set up correctly, there shouldn not be any error/warning message.

## Task7: Client - Test the HTTPS Connection

Since the server certificate is based on the self-signed root certificate from RootCA node. To successfully test the HTTPS connection, the premise here is that the client should trust that self-signed certificate. Therefore, it is necessary to pass in the root certificate as a parameter to the client node before connecting to the server!

As demonstrated in the previous steps, here you need to install the necessary packages and libraries, use the Infura API as the gateway to connect to Ethereum Ropsten network, read the `rootCert` variable from the deployed smart contract and write it to the local client node. So eventually a root certificate should be saved on the `Client` node.

After getting the root certificate `root_cert.crt`, let's first modify the host file such that the domain name can correctly map to the IP address of the server machine. Open the **/etc/hosts** file as root user with your favorite text editor, and add the following line below the existing contents. (Hint: you can get the IP of the server by using the command `$ hostname -i` on the `server` node.)
`{SERVER_IP_ADDRESS} jhuisi-ethcert-server.com`

<img src="https://i.imgur.com/6eVgqGa.png" alt="drawing" width="500"/>

Now, it's time to test the HTTPS connection! Use both the `curl` and `openssl s_client` command to verify that the HTTPS connection works out.

```bash
$ curl --cacert root_cert.crt https://jhuisi-ethcert-server.com

# `-showcerts` flag will print out all certificates in the certificate chain
# specify the root certificate using the `-CAfile` flag
$ openssl s_client -showcerts -connect {SERVER_IP}:443 -CAfile root_cert.crt

# ex: $ openssl s_client -showcerts -connect 172.17.1.1:443 -CAfile root_cert.crt
```

## References

- https://github.com/ethereumjs/ethereumjs-tx
