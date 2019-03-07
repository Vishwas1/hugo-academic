+++
title = "Stellar private network setup and wallet app tutorial"
subtitle = ""
summary = ""
date = 2019-03-07T10:00:00+05:30
draft = false
authors = ["Vishwas Anand"]
featured = false
tags = ["blockchain", "ethereum", "stellar","wallet", "dapps"]
categories = ["Blockchain"]
# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder. 
[image]
  # Caption (optional)
  caption = ""

  # Focal point (optional)
  # Options: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight
  focal_point = ""
+++

[Stellar][] - the blockchain network based on a federated consensus
protocol - is cool but it comes with a lot of complexities. However, as far as
I researched, I found it's not as stable as Ethereum. Builds have bugs (check
out the issue list [here](https://github.com/stellar/stellar-core/issues)) but
the good news is, they have very active and well-managed support. A separate
[forum](https://galactictalk.org/) is there for that. You can ask or checkout
questions there.

If you want to understand the core concept that lies behind the [Stellar][]
then you can check out their docs
[here](https://www.stellar.org/developers/guides/get-started/),
[here](https://www.stellar.org/how-it-works/stellar-basics/) and
[here](https://www.stellar.org/how-it-works/stellar-basics/explainers/). If you
want to understand more, like how its consensus works then its better to take a
look into its
[whitepaper](https://www.stellar.org/papers/stellar-consensus-protocol.pdf).
Developers who want to build applications on top of Stellar's public or test
network might want to take a look at
[this](https://www.stellar.org/developers/guides/get-started/)  and
[this](https://www.stellar.org/developers/reference/) link.

[Stellar]: https://www.stellar.org

## Objective of this doc

Recently, I got a chance to explore **Stellar**. I am coming from Ethereum and
Bitcoin background. Which means that I have considerable amount of
understanding of how these public chains work or even how blockchain technology
works under the hood. Having said that, let us admit one thing, that Stellar
network setup is complex and is in the development phase so unstable too. It
has several different components, primarily Stellar-Core, Horizon, Stellar-SDK
etc, unlike Ethereum which has just Geth and Web3. I think this is the reason
that there is no single documentation (consolidated one) out there on the web
for setting up the Stellar private network (or at least I could not find one)

> Folks, if you haven't explored Ethereum or Bitcoin as of now, then I would
> suggest you **NOT** to explore Stellar first. 

My primary goal in this blog to *simplify the process of Stellar network
setup*. I will also try to compare the installation with Geth so that people
coming from an Ethereum background can understand it better. Also to make thing
simple, I won't talk about the concepts of Quorum or Quorum slices.

Before we go forward let me tell you few disclaimers :
- All setups are done on Ubuntu - 18.04 system.
- These are the basic configurations, if you want complexities you can checkout
  official docs.
- Codes written for wallet app may NOT be optimised and can have security
  issues.

## Basic Architecture

![arch](Stellar-basic-arch.jpg)

Let us understand the basic architecture of Stellar. The core component of the
Stellar network is the **Stellar-Core** which is the backbone of the Stellar
network. Its work is to validate and agree with other instance of Core on the
status of every transaction through the [Stellar consensus
protocol(SCP)](https://www.stellar.org/developers/guides/concepts/scp.html).
The network itself is a collection of Stellar Cores run by various individuals.

Some instances have a **Stellar-Horizon** server(a RESTful HTTP API server) you
can communicate with, while others exist only to add reliability to the overall
network, which means that not all cores must have the Horizon server.
Applications interact with the Stellar network through Horizon servers using
**Stellar-SDK**.

Applications connect with the Horizon server using javascript, java, Go-based
SDKs, called **Stellar-SDK**. 

Stellar uses a relational database system,
[Postgres](https://www.postgresql.org/), for storing data. **Stellar-Core** and
**Stellar-Horizon**  will have separate databases.

## What we will do exactly?

We will divide our goal into 2 parts:
1. Setup a private network with 2 nodes.
2. Implement a simple Node Js wallet app to do some transactions.

## Setup Private Network

- Installations
    - Install Stellar-Core
    - Install Horizon Server
    - Install Postgres DB
- Setups
    - We will setup node01, a Stellar-Core along with its Postgres DB,
      `stellar_node01_db`.
    - We will setup node02, a Stellar-Core along with its Postgres DB,
      `stellar_node02_db`.
    - We will change the configuration files of both nodes to make them peers.  
    - We will set up Horizon server along with its Postgres DB,
      `stellar_horizon01_db` and will connect it to the node01 core. After this
      setup, we will get an Horizon-endpoint for our SDK to connect with.


### Install Stellar-Core

The first thing we will do is to install Stellar-Core. This is most time taking
part and I suggest you do it very carefully otherwise you might end up wasting
time. 

Follow these steps:

```
git clone --single-branch --branch prod https://github.com/stellar/stellar-core.git
cd stellar-core
git submodule init
git submodule update
./autogen.sh
./configure
make
sudo make install
stellar-core --version # to verify the installation
```

Basically what we are doing here, cloning the repo (the prod one not the
master) and building the source code and then installing it. 

### Install Horizon server

- Go to [this](https://github.com/stellar/horizon/releases) link and download
  the latest version of Horizon
- Extract it and place it somewhere (I usually put software inside `/tools` you
  can put where ever you want)
- Update the `./profile` file in `$HOME`
    - `export PATH=$PATH:/tools/horizon` you can do according to your
      configuration. 
- In the terminal type `horizon version` to verify the installation.

### Install Postgres DB

- `sudo apt install postgresql postgresql-contrib`

Follow [this](http://postgresguide.com/setup/install.html) doc for the Postgres
DB installation. 


### Setup node01

1. Create a folder called `node01`
2. cd into it and create a file `touch stellar-core.cfg`
3. Open the file and paste the code below `vim stellar-core.cfg`

    ```
    # What port stellar-core listens for commands on. This is for Horizon server.
    HTTP_PORT=11626

    PUBLIC_HTTP_PORT=false

    # If it is true, It prevents you from trying to connect to other peers
    RUN_STANDALONE=false

    # A phrase for your network. All nodes should have the same network phrase.
    NETWORK_PASSPHRASE="Standalone Pramati Network ; Oct 2018"

    # The seed used for generating the public key this node will be identified within SCP.
    NODE_SEED="XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX self"

    # Only nodes that want to participate in SCP should set NODE_IS_VALIDATOR=true.
    # Most instances should operate in observer mode with NODE_IS_VALIDATOR=false.
    NODE_IS_VALIDATOR=true

    # Comma separated peers list
    KNOWN_PEERS=["127.0.0.1:11635"]

    # Postgres DB URL
    DATABASE="postgresql://dbname=stellar_node01_db host=localhost user=your_user_name password=your_user_password"

    # The port other instances of stellar-core can connect to you on.
    PEER_PORT=11625

    # Log level setup
    COMMANDS=["ll?level=trace"]

    FAILURE_SAFETY=0
    UNSAFE_QUORUM=true

    #The public keys of the Stellar servers
    [QUORUM_SET]
    THRESHOLD_PERCENT=100

    # comma sepearted validator list 
    VALIDATORS=["$self"]

    [HISTORY.vs]
    get="cp /tmp/stellar-core/history/vs/{0} {1}"
    put="cp {0} /tmp/stellar-core/history/vs/{1}"
    mkdir="mkdir -p /tmp/stellar-core/history/vs/{0}"
    ```

    The above is minimum configuration needed. Most of parameters are self
    explanatory. Full details of all configuration can be found
    [here](https://github.com/stellar/stellar-core/blob/master/docs/stellar-core_example.cfg)

4. Let's create the seed phrase for this network.
    - `stellar-core --genseed`
    - Copy the `Secret seed` and paste into `NODE_SEED` in the configuration
      file.

5. Set up the database for node01.
    - `sudo su postgres`
    - `psql`
    - Create user `CREATE USER someuser WITH PASSWORD 'somepass';`
    - Give him admin access `ALTER USER someuser WITH SUPERUSER;`
    - Create database for node01 `CREATE DATABASE stellar_node01_db;`
    - Grant permission to this user in database `GRANT ALL PRIVILEGES ON
      DATABASE stellar_node01_db TO stellar_node_user;`
    - Set the `DATABASE` url (db, host, username, password) in the
      configuration file.
6. Set `PEER_PORT` and `HTTP_PORT` as well.
7. Set `NETWORK_PASSPHRASE`. 
8. Notice that in `KNOWN_PEERS` array, I have added a url `127.0.0.1:11635`, I
   am going to set `PEER_PORT` value for node02 as `11635`.
9. Initliaze database -  basically creating meta data.
    - `stellar-core --newdb`
    - Notice that it gives us `Root account seed`. Just make a note of it
      somewhere, we will have to use the root account later.
10. Set a flag to force the node to start SCP immediately.
    - `stellar-core --forcescp`
11. Run the node01
    - `stellar-core`


### Setup node02

1. All steps are same as of node01 but change the values of these parameters in
node02's config file:
    - `PEER_PORT` : 11635
    - `HTTP_PORT` : 11636
    - `KNOWN_PEERS` : KNOWN_PEERS=["127.0.0.1:11625"]
    - `NODE_SEED`
2. Create database `stellar_node02_db` and grant access to the user which we
created earlier.
3. Set the `DATABASE` url (db, host, username, password) in the configuration
file.
4. Run the node `stellar-core`

Note*: Make sure to put the value of `NETWORK_PASSPHRASE` for node02 same as
node01. Also notice `Root Account` created after running command `stellar-core --newdb`
for this node, its same as of the 1st one and I do not know the reason
for that. 

### Making them peers

After doing the above steps, they should become peers. The peer configuration
is done in `KNOWN_PEERS` parameter in the config file. Notice I have exchanged
the urls of nodes in here. Also, you can check if nodes are synced by running
command `stellar-core --c 'info'` 

### Setup Horizon server

1. Create database `stellar_horizon_db` and grant access to the user which we created earlier.
2. Initialise Horizon DB 
    - `DATABASE_URL="postgresql://someuser:somepass@localhost:5432/stellar_horizon_db" horizon db init`
3. Run Horizon server 
    ```
    horizon \
        --port 8000 \
        --ingest=true \
        --db-url "postgresql://someuser:somepass@localhost:5432/stellar_horizon_db" \
        --stellar-core-db-url "postgresql://someuser:somepass@localhost:5432/stellar_node01_db" \
        --stellar-core-url "http://127.0.0.1:11626"
    ```
    Remember, we want to connect this horizon server with node01. That is why we are giving http url (`HTTP_PORT` we set in config of node01) and db-url for node01.
4. Horizon endpoint to connect with : `http://127.0.0.1:8000` - you can browse this URL to verify that its connected to same network or not.

> In this way we will have a proper private network setup with 2 nodes running
> (self-validating) and one REST server running. Now we will move towards
> developing our wallet app to do some transactions.

## Developing wallet app

This wallet is going to be a simple application built on `NodeJS` using
`Express`, I hope you have basic knowledge of this technology. Basically, it's
going to be APIs which will interact with **Stellar-Horizon** server within
turns, is interacting with the blockchain.

Steps which we will follow : 
1. Create 2 accounts. (API: `/newAccount`)
2. Get some money from `Root Account` in both of these accounts. (API: `/faucet`)
3. Check the balance of accounts. (API: `/balance`)
4. Send some money from account01 to account02. (API: `/payment`) 

For implementing these APIs we are going to make use of **stellar-SDK**, which
is a javascript library. 
- Create a folder `mkdir stellar-wallet` 
- cd into it.
- `npm init`
- `touch index.js`
- Copy paste this code
- `npm i`
- Run the app `node index.js`

```js

const express = require('express')
const bodyParser = require('body-parser')
const rp = require('request-promise')
const port = process.env.PORT || 4000
const app = express()
const Stellar = require('stellar-sdk')

app.use(bodyParser.json())
app.use(bodyParser.urlencoded({ extended: true })) 

const HORIZON_ENDPOINT = 'http://127.0.0.1:8000'
const NETWORK_PASSPHRASE = "Standalone Pramati Network ; Oct 2018"

// Getting instance of Stellar blockchain
Stellar.Network.use(new Stellar.Network(NETWORK_PASSPHRASE));
var opts = new Stellar.Config.setAllowHttp(true);
var server = new Stellar.Server(HORIZON_ENDPOINT, opts);

let accounts = []

// Creating new account 
const creatingAccount = async (req,res) =>{
    try{
        console.log(`creatingAccount method got called`)
        let pair = Stellar.Keypair.random()
        let account = {
            pk : pair.publicKey(),
            sk : pair.secret()
        }
        accounts.push(account)
        res.send(account);
    }catch(err){
        res.send({"Msg" : "ERROR : " + err})
    }
}

// Get 100 coins from root account
const getFromFaucet = async (req,res) =>{
    try{
        const pk = req.body.pk
        if(pk){
            // faucet is our root account. Make sure you replace this value with your key
            let sourceKeys = Stellar.Keypair.fromSecret("SDJ5AQWLIAYT22TCYSKOQALI3SNUMPAR63SEL73ASALDP6PYDN54FARM");
            // loading root account
            server.loadAccount(sourceKeys.publicKey())
            .then(function(sourceAccount) {
                let txn = new Stellar.TransactionBuilder(sourceAccount)
                            .addOperation(
                                Stellar.Operation.createAccount({
                                destination: pk,
                                startingBalance: "100"}))
                            .addMemo(Stellar.Memo.text('Test Transaction'))
                            .build();
                txn.sign(sourceKeys);
                return server.submitTransaction(txn);
                })
            .then(function(result) {
                res.send({"Msg" : `SUCCESS : ${JSON.stringify(result)}`})
            })
            .catch(function(error) {
                console.error('Something went wrong!', error);
                res.send({"Msg" : `ERROR : ${error}`})
            });
        }else{
            res.send({"Msg" : "ERROR : please provide public key!"})
        }
    }catch(err){
        res.send({"Msg" : `ERROR : ${error}`})
    }
}

// Fetch all created accounts
const getAccounts =  async (req,res) =>{
    res.send(accounts);
}

// Get balance of an account
const getBalance = async (req, res) =>{
    try{
        const pk = req.body.pk;
        let balance = 0;
        // Load newly created accounts
        account = await server.loadAccount(pk)
        // check the balances
        account.balances.forEach((bal) => {
            balance = balance + bal.balance;
        })
        res.send({"Msg" : balance})
    }catch(err){
        res.send({"Msg" : "ERROR : " + err})
    }    
}
                
// Do transactions
const makePayment = async (req,res) => {

    const {from, to, value} =  req.body;
    //Let get the secret of the spender
    const spender = accounts.find((acc) => {
        if(acc.pk === from) return acc;
    })
    if(spender && spender != null){
        // First, check to make sure that the destination account exists.
        // You could skip this, but if the account does not exist, you will be charged
        // the transaction fee when the transaction fails.
        server.loadAccount(to)
        .catch((err)=>{
            res.send({"Msg" : `Error : receiever ${to} not found!`})
        })
        .then(() =>{
            // lets load spender account
            return server.loadAccount(from);
        })
        .then((spenderAccount) => {
            // Start building the transaction.
            const transaction = new Stellar.TransactionBuilder(spenderAccount)
            .addOperation(Stellar.Operation.payment({
                destination: to,
                // Because Stellar allows transaction in many currencies, you must
                // specify the asset type. The special "native" asset represents Lumens.
                asset: Stellar.Asset.native(),
                amount: value
            }))
            // A memo allows you to add your own metadata to a transaction. It's
            // optional and does not affect how Stellar treats the transaction.
            .addMemo(Stellar.Memo.text('Test Transaction'))
            .build()
            // get the key pair for signing the transaction
            const pairA =  Stellar.Keypair.fromSecret(spender.sk);
            // Sign the transaction to prove you are actually the person sending it
            transaction.sign(pairA)
            return server.submitTransaction(transaction);
        })
        .then((result)=>{
            res.send({"Msg" : JSON.stringify(result, null, 2)})
        })
        .catch((err)=>{
            res.send({"Msg" : `Error : Somethis went wrong : ${JSON.stringify(err.response.data.extras)}`})
        })
    }else{
        res.send({"Msg" : `Error : spender  ${to} not found!`})
    }
}
/* CORS */
app.use((req, res, next) => {
    // Website you wish to allow to connect
    res.setHeader('Access-Control-Allow-Origin', '*')
  
    // Request methods you wish to allow
    res.setHeader('Access-Control-Allow-Methods', 'GET, POST, OPTIONS, PUT, PATCH, DELETE')
  
    // Request headers you wish to allow
    res.setHeader('Access-Control-Allow-Headers', 'Origin,X-Requested-With,content-type')
  
    // Pass to next layer of middleware
    next()
})

/* API Routes */
app.get('/newAccount', creatingAccount)
app.get('/accounts', getAccounts)
app.post('/faucet',getFromFaucet)
app.post('/balance', getBalance)
app.post('/payment', makePayment)

/* Serve API */
app.listen(port, () => {
  console.log(`Stellar test app listening on port ${port}!`)
})
```

You must be wondering why I called `/faucet` API for both of the accounts. Only
one-time call would be enough right? The problem is when you create a new
key-pair, blockchain does not know about it unless until you call
`Stellar.Operation.createAccount`  operation, which we are doing it in
`/faucet` API while building transaction. Of course we can handle this problem
in a better way but right now I too lazy to do so :D That is why for both of
the accounts you have to call `/faucet` API to register the account in the
blockchain so that next time when you do `payment` blockchain recognises these
accounts and does not throw error. Also, do not forget to replace `root
account` in `getFromFaucet()`


Full list of API calls are here:

```
# APIs

/* API Routes */
app.get('/newAccount', creatingAccount)
app.get('/accounts', getAccounts)
app.post('/faucet',getFromFaucet)
app.post('/balance', getBalance)
app.post('/payment', makePayment)

# New Account

Method : GET
URL : http://localhost:4000/newAccount  
Response : 
{
    "pk": "GBVLTVGDDO3CAEZQQEDBZCTHZKXZUT7LDIGGXD7W5R2YKOQLPFMV3WLK",
    "sk": "SBYHYPWNC5ZVWPH73VHBVWF2D4PDXV4466GLL2WQNZQKHQTJJSQPGZBM"
}

# All Accounts

Method : GET
URL : http://localhost:4000/accounts
Response : 
[
    {
        "pk": "GCMIEBN26ACIUJVBGB3XT2ZAGIPZTG3SCV5K2FCBUY2AMYGBOOKOF4P6",
        "sk": "SB6DP7TKZW777PGOC7IATGHQ52SBS4SCSKTSNWA2VTBXJG5YERYXL3ZQ"
    },
    {
        "pk": "GCVVJY5EXBTKIG76PFZCULLIWJGAYBTHGBPQVRWZHJC7FGUJMSYVZ675",
        "sk": "SA5OOHTOTO3RSTRZ2ZZLVXHTP2AASV3P22ZP44CURJMBLTXUGVZRKWTH"
    }
]

# Earn money from faucet

Method : POST
URL : http://localhost:4000/faucet
Format : JSON
Request : 
{
	"pk" : "GCVVJY5EXBTKIG76PFZCULLIWJGAYBTHGBPQVRWZHJC7FGUJMSYVZ675"
}
Response : 
{
    "Msg": "010000.0000000"
}

# Get balance of an account

Method : POST
URL : http://localhost:4000/balance
Format : JSON
Request : 
{
	"pk" : "GCVVJY5EXBTKIG76PFZCULLIWJGAYBTHGBPQVRWZHJC7FGUJMSYVZ675"
}
Response : 
{
    "Msg": "010000.0000000"
}

# Make payment 

Method : POST
URL : http://localhost:4000/payment
Format : JSON
Request : 
{
	"from" : "GCMIEBN26ACIUJVBGB3XT2ZAGIPZTG3SCV5K2FCBUY2AMYGBOOKOF4P6",
	"to" : "GCVVJY5EXBTKIG76PFZCULLIWJGAYBTHGBPQVRWZHJC7FGUJMSYVZ675",
	"value" : "20"
}
Response : 
{
    "Msg": "{\n  \"_links\": {\n    \"transaction\": {\n      \"href\": \"http://172.18.0.2:8000/transactions/f9c2425af28995d8ba0f8751fe8a0c5c77f7388b768cfc28b1206d2a8fce346e\"\n    }\n  },\n  \"hash\": \"f9c2425af28995d8ba0f8751fe8a0c5c77f7388b768cfc28b1206d2a8fce346e\",\n  \"ledger\": 18454,\n  \"envelope_xdr\": \"AAAAAJiCBbrwBIomoTB3eesgMh+Zm3IVeq0UQaY0BmDBc5TiAAAAZAAASA8AAAABAAAAAAAAAAEAAAAQVGVzdCBUcmFuc2FjdGlvbgAAAAEAAAAAAAAAAQAAAACrVOOkuGakG/55ciotaLJMDAZnMF8Kxtk6RfKaiWSxXAAAAAAAAAAAC+vCAAAAAAAAAAABwXOU4gAAAEA6DJsf9db07hMIZ8FhYQ+o5ptErY8OaDwBBw+RopZO2xfP+dwtxKqT/ubPocWvyObAkJCGRFS2EA59rdCKEzQD\",\n  \"result_xdr\": \"AAAAAAAAAGQAAAAAAAAAAQAAAAAAAAABAAAAAAAAAAA=\",\n  \"result_meta_xdr\": \"AAAAAQAAAAIAAAADAABIFgAAAAAAAAAAmIIFuvAEiiahMHd56yAyH5mbchV6rRRBpjQGYMFzlOIAAAAXSHbnnAAASA8AAAAAAAAAAAAAAAAAAAAAAAAAAAEAAAAAAAAAAAAAAAAAAAAAAAABAABIFgAAAAAAAAAAmIIFuvAEiiahMHd56yAyH5mbchV6rRRBpjQGYMFzlOIAAAAXSHbnnAAASA8AAAABAAAAAAAAAAAAAAAAAAAAAAEAAAAAAAAAAAAAAAAAAAAAAAABAAAABAAAAAMAAEgWAAAAAAAAAACYggW68ASKJqEwd3nrIDIfmZtyFXqtFEGmNAZgwXOU4gAAABdIduecAABIDwAAAAEAAAAAAAAAAAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAEAAEgWAAAAAAAAAACYggW68ASKJqEwd3nrIDIfmZtyFXqtFEGmNAZgwXOU4gAAABc8iyWcAABIDwAAAAEAAAAAAAAAAAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAMAAEgSAAAAAAAAAACrVOOkuGakG/55ciotaLJMDAZnMF8Kxtk6RfKaiWSxXAAAABdIdugAAABIEgAAAAAAAAAAAAAAAAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAEAAEgWAAAAAAAAAACrVOOkuGakG/55ciotaLJMDAZnMF8Kxtk6RfKaiWSxXAAAABdUYqoAAABIEgAAAAAAAAAAAAAAAAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAA==\"\n}"
}
```

Note*: You can follow these links for references.
- [create accounts](https://www.stellar.org/developers/guides/get-started/create-account.html)
- [key-pair](https://stellar.github.io/js-stellar-sdk/Keypair.html)
- [friendbot-for-faucet](https://github.com/stellar/go/tree/master/services/friendbot)
- [transactions](https://www.stellar.org/developers/guides/get-started/transactions.html)


## Stellar Laboratory

Do not want to go through pain of implementing the Wallet-app? Fine! You can
work on and test out with your private network using [**Stellar
Laboratory**](https://www.stellar.org/laboratory/)

You just have to click on **custom** button in top-right and configure your
network (basically horizon endpoint) there and start playing around.

## Stellar Explorer

If you want to setup a proper user interface for your network to view
transactions , operations, ledger etc., you can do that using [**Stellar
Explorer**](https://github.com/chatch/stellarexplorer). Its similar to [**Ether
Scan**](https://etherscan.io/) for Ethereum.

Steps to setup:
- `git clone https://github.com/chatch/stellarexplorer`
- `cd` into directory.
- `npm i`
- `npm start`
- This app will automatically connect to your horizon endpoint
  (http://localhost:8000 by default). If you want to change the endpoint, then
  click on **Set Custom Network** button and configure.


----

Well, there could be many disclaimers in this tutorials but let's overlook
those to learn just the basics of the Stellar blockchain. I hope this tutorial
would definitely help you out getting started with Stellar. Cheers!


*The blog was originally published [here](https://labs.imaginea.com/post/stellar-bc-wallet/)*