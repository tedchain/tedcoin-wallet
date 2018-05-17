# Tedcoin Wallet

The dev server by default expects tedcoin to run on http://localhost:3001

## Build Setup


``` bash
# install dependencies
npm install

# serve with hot reload at localhost:8080
npm run dev

# build for production with minification
npm run build
```

# ABOUT TEDCOIN PROJECT
## 1) INTRODUCTION
In this tutorial we will code from scratch some of the basic concepts that are needed for a working cryptocurrency. The angle is always to implement things in the most simplest way.

The project that we will build in this tutorial is called "Tedcoin". The programming language is Typescript. The Tedcoin is in some terms an extension to the [TedChain](https://github.com/tedchain/tedchain).

The final version of Tedcoin is by no means a "production ready" implementation of a cryptocurrency, but rather tries to show that the basic principles in a cryptocurrency can be implemented in a concise way.

I hope this tutorial will help you understand more of the technical aspects of a cryptocurrency, and our project.

## 2) MINIMAL WORKING BLOCKCHAIN
### Overview
The basic concept of blockchain is quite simple: a distributed database that maintains a continuously growing list of ordered records. We will implement toy version of such blockchain. We will have the following basic functionalities of blockchain:
* A defined block and blockchain structure
* Methods to add new blocks to the blockchain with arbitrary data
* Blockchain nodes that communicate and sync the blockchain with other nodes
* A simple HTTP API to control the node

### Block structure
We will start by defining the block structure. Only the most essential properties are included at the block at this point.
* Index : The height of the block in the blockchain
* Data: Any data that is included in the block.
* Timestamp: A timestamp
* Hash: A sha256 hash taken from the content of the block
* PreviousHash: A reference to the hash of the previous block. This value explicitly defines the previous block.

![alt tag](https://i.imgur.com/IjsVNeQ.png)

The code for the block structure looks like the following:
```
class Block {

    public index: number;
    public hash: string;
    public previousHash: string;
    public timestamp: number;
    public data: string;

    constructor(index: number, hash: string, previousHash: string, timestamp: number, data: string) {
        this.index = index;
        this.previousHash = previousHash;
        this.timestamp = timestamp;
        this.data = data;
        this.hash = hash;
    }
}
```

### Block Hash
The block hash is one of the most important property of the block. The hash is calculated over all data of the block. This means that if anything in the block changes, the original hash is no longer valid. The block hash can also be thought as the unique identifier of the block. For instance, blocks with same index can appear, but they all have unique hashes.

We calculate the hash of the block using the following code:
```
const calculateHash = (index: number, previousHash: string, timestamp: number, data: string): string =>
CryptoJS.SHA256(index + previousHash + timestamp + data).toString();
```

It should be noted that the block hash has not yet nothing to do with mining, as there is no proof-of-work problem to solve. We use block hashes to preserve integrity of the block and to explicitly reference the previous block.

An important consequence of the properties hash and previousHash is that a block can't be modified without changing the hash of every consecutive block.

This is demonstrated in the example below. If the data in block 44 is changed from "DESERT" to "STREET", all hashes of the consecutive blocks must be changed. This is because the hash of the block depends on the value of the previousHash (among other things).

![alt tag](https://i.imgur.com/SK8WPum.png)

### Genesis block
Genesis block is the first block in the blockchain. It is the only block that has no previousHash. We will hard code the genesis block to the source code:
```
const genesisBlock: Block = new Block(
    0, '816534932c2b7154836da6afc367695e6337db8a921823784c14378abed4f7d7', null, 1465154705, 'my genesis block!!'
);
```

### Generating a block
To generate a block we must know the hash of the previous block and create the rest of the required content (= index, hash, data and timestamp). Block data is something that is provided by the end-user but the rest of the parameters will be generated using the following code:
```
const generateNextBlock = (blockData: string) => {
    const previousBlock: Block = getLatestBlock();
    const nextIndex: number = previousBlock.index + 1;
    const nextTimestamp: number = new Date().getTime() / 1000;
    const nextHash: string = calculateHash(nextIndex, previousBlock.hash, nextTimestamp, blockData);
    const newBlock: Block = new Block(nextIndex, nextHash, previousBlock.hash, nextTimestamp, blockData);
    return newBlock;
};
```

### Storing the blockchain
For now we will only use an in-memory Javascript array to store the blockchain. This means that the data will not be persisted when the node is terminated.
```
const blockchain: Block[] = [genesisBlock];
```

### Validating the integrity of blocks
At any given time we must be able to validate if a block or a chain of blocks are valid in terms of integrity. This is true especially when we receive new blocks from other nodes and must decide whether to accept them or not.
* For a block to be valid the following must apply:
* The index of the block must be one number larger than the previous
* The previousHash of the block match the hash of the previous block
* The hash of the block itself must be valid

This is demonstrated with the following code:
```
const isValidNewBlock = (newBlock: Block, previousBlock: Block) => {
    if (previousBlock.index + 1 !== newBlock.index) {
        console.log('invalid index');
        return false;
    } else if (previousBlock.hash !== newBlock.previousHash) {
        console.log('invalid previoushash');
        return false;
    } else if (calculateHashForBlock(newBlock) !== newBlock.hash) {
        console.log(typeof (newBlock.hash) + ' ' + typeof calculateHashForBlock(newBlock));
        console.log('invalid hash: ' + calculateHashForBlock(newBlock) + ' ' + newBlock.hash);
        return false;
    }
    return true;
};
```

We must also validate the structure of the block, so that malformed content sent by a peer won’t crash our node.
```
const isValidBlockStructure = (block: Block): boolean => {
    return typeof block.index === 'number'
        && typeof block.hash === 'string'
        && typeof block.previousHash === 'string'
        && typeof block.timestamp === 'number'
        && typeof block.data === 'string';
};
```

Now that we have a means to validate a single block we can move on to validate a full chain of blocks. We first check that the first block in the chain matches with the genesisBlock. After that we validate every consecutive block using the previously described methods. This is demostrated using the following code:
```
const isValidChain = (blockchainToValidate: Block[]): boolean => {
    const isValidGenesis = (block: Block): boolean => {
        return JSON.stringify(block) === JSON.stringify(genesisBlock);
    };

    if (!isValidGenesis(blockchainToValidate[0])) {
        return false;
    }

    for (let i = 1; i < blockchainToValidate.length; i++) {
        if (!isValidNewBlock(blockchainToValidate[i], blockchainToValidate[i - 1])) {
            return false;
        }
    }
    return true;
};
```

### Choosing the longest chain
There should always be only one explicit set of blocks in the chain at a given time. In case of conflicts (e.g. two nodes both generate block number 72) we choose the chain that has the longest number of blocks. In the below example, the data introduced in block 72: a350235b00 will not be included in the blockchain, since it will be overridden by the longer chain.

![alt tag](https://i.imgur.com/fFSf72w.png)

This is logic is implemented using the following code:
```
const replaceChain = (newBlocks: Block[]) => {
    if (isValidChain(newBlocks) && newBlocks.length > getBlockchain().length) {
        console.log('Received blockchain is valid. Replacing current blockchain with received blockchain');
        blockchain = newBlocks;
        broadcastLatest();
    } else {
        console.log('Received blockchain invalid');
    }
};
```

### Communicating with other nodes
An essential part of a node is to share and sync the blockchain with other nodes. The following rules are used to keep the network in sync.
* When a node generates a new block, it broadcasts it to the network
* When a node connects to a new peer it querys for the latest block
* When a node encounters a block that has an index larger than the current known block, it either adds the block the its current chain or querys for the full blockchain.

![alt tag](https://i.imgur.com/x8OptmR.png)

We will use websockets for the peer-to-peer communication. The active sockets for each nodes are stored in the const sockets: WebSocket[] variable. No automatic peer discovery is used. The locations (= Websocket URLs) of the peers must be manually added.

### Controlling the node
The user must be able to control the node in some way. This is done by setting up a HTTP server.

```
const initHttpServer = ( myHttpPort: number ) => {
    const app = express();
    app.use(bodyParser.json());

    app.get('/blocks', (req, res) => {
        res.send(getBlockchain());
    });
    app.post('/mineBlock', (req, res) => {
        const newBlock: Block = generateNextBlock(req.body.data);
        res.send(newBlock);
    });
    app.get('/peers', (req, res) => {
        res.send(getSockets().map(( s: any ) => s._socket.remoteAddress + ':' + s._socket.remotePort));
    });
    app.post('/addPeer', (req, res) => {
        connectToPeers(req.body.peer);
        res.send();
    });

    app.listen(myHttpPort, () => {
        console.log('Listening http on port: ' + myHttpPort);
    });
};
```

As seen, the user is able to interact with the node in the following ways:
* List all blocks
* Create a new block with a content given by the user
* List or add peers

The most straightforward way to control the node is e.g. with Curl:
```
#get all blocks from the node
> curl http://localhost:3001/blocks
```

### Architecture
It should be noted that the node actually exposes two web servers: One for the user to control the node (HTTP server) and one for the peer-to-peer communication between the nodes. (Websocket HTTP server)

![alt tag](https://i.imgur.com/KKrt7qK.png)

### Conclusions
Tedcoin is for now a just toy "general purpose" blockchain. Moreover, we will show how some of the basic principles of blockchain can be implemented in quite a simple way. We will add the proof-of work algorithm (mining) to the Tedcoin.

## 3) PROOF OF WORK
### Overview
We will implement a simple Proof-of-Work scheme to the toy blockchain version. Anyone could add a block to the chain without a cost. With Proof-of-work we introduce a computational puzzle that needs to be solved, before a block can be added to the blockchain. Trying to solve this puzzle is commonly known as "mining".

With Proof-of-work we also can control (approximately) the interval on how often a block is introduced to the blockchain. This is done by changing the difficulty of the puzzle. If blocks are mined too often, the difficulty of the puzzle will increase and vice versa.

It should be noted that we do not yet introduce transactions. This means there is actually no incentive for the miners to generate a block. Generally in cryptocurrencies, the miner is rewarded for finding a block, but this is not the case yet in our blockchain.

### Difficulty, nonce and the proof-of-work puzzle
We will add two new properties to the block structure: difficulty and nonce. To understand the meaning of those, we must first introduce the Proof-of-work puzzle.

The Proof-of-work puzzle is to find a block hash, that has a specific number of zeros prefixing it. The difficulty property defines how many prefixing zeros the block hash must have, in order for the block to be valid. The prefixing zeros are checked from the binary format of the hash.

Below are some examples of valid and non-valid hashes for various difficulties:

![alt tag](https://i.imgur.com/yjdWoiy.png)

The code that checks that the hash is correct in terms of difficulty:
```
const hashMatchesDifficulty = (hash: string, difficulty: number): boolean => {
    const hashInBinary: string = hexToBinary(hash);
    const requiredPrefix: string = '0'.repeat(difficulty);
    return hashInBinary.startsWith(requiredPrefix);
};
```

In order to find a hash that satisfies the difficulty, we must be able to calculate different hashes for the same content of the block. This is done by modifying the nonce parameter. Because SHA256 is a hash function, each time anything in the block changes, the hash will be completely different. "Mining" is basically just trying a different nonce until the block hash matches the difficulty.

Now that the difficulty and nonce are added, the block structure looks like this:
```
class Block {
    public index: number;
    public hash: string;
    public previousHash: string;
    public timestamp: number;
    public data: string;
    public difficulty: number;
    public nonce: number;
    constructor(index: number, hash: string, previousHash: string,
                timestamp: number, data: string, difficulty: number, nonce: number) {
        this.index = index;
        this.previousHash = previousHash;
        this.timestamp = timestamp;
        this.data = data;
        this.hash = hash;
        this.difficulty = difficulty;
        this.nonce = nonce;
    }
}
```

We must also remember to update the genesis block!

### Finding a block
As described above, to find a valid block hash we must increase the nonce as until we get a valid hash. To find a satisfying hash is completely a random process. We must just loop through enough nonces until we find a satisfying hash:
```
const findBlock = (index: number, previousHash: string, timestamp: number, data: string, difficulty: number): Block => {
    let nonce = 0;
    while (true) {
        const hash: string = calculateHash(index, previousHash, timestamp, data, difficulty, nonce);
        if (hashMatchesDifficulty(hash, difficulty)) {
            return new Block(index, hash, previousHash, timestamp, data, difficulty, nonce);
        }
        nonce++;
    }
};
```

When the block is found, it is broadcasted to the network.

### Consensus on the difficulty
We have now the means to find and verify the hash for a given difficulty, but how is the difficulty determined? There must be a way for the nodes to agree what the current difficulty is. For this we introduce some new rules that we use to calculate the current difficulty of the network.

Lets define the following new constants for the network:
* BLOCK_GENERATION_INTERVAL, defines how often a block should be found. (in Bitcoin this value is 10 minutes)
* DIFFICULTY_ADJUSTMENT_INTERVAL, defines how often the difficulty should adjust to the increasing or decreasing network hashrate. (in Bitcoin this value is 2016 blocks)

We will set the block generation interval to 10s and difficulty adjustment to 10 blocks. These constants do not change over time and they are hard coded.
```
// in seconds
const BLOCK_GENERATION_INTERVAL: number = 10;

// in blocks
const DIFFICULTY_ADJUSTMENT_INTERVAL: number = 10;
```

Now we have the means to agree on a difficulty of the block. For every 10 blocks that is generated, we check if the time that took to generate those blocks are larger or smaller than the expected time. The expected time is calculated like this: BLOCK_GENERATION_INTERVAL * DIFFICULTY_ADJUSTMENT_INTERVAL. The expected time represents the case where the hashrate matches exactly the current difficulty.

We either increase or decrease the difficulty by one if the time taken is at least two times greater or smaller than the expected difficulty. The difficulty adjustment is handled by the following code:
```
const getDifficulty = (aBlockchain: Block[]): number => {
    const latestBlock: Block = aBlockchain[blockchain.length - 1];
    if (latestBlock.index % DIFFICULTY_ADJUSTMENT_INTERVAL === 0 && latestBlock.index !== 0) {
        return getAdjustedDifficulty(latestBlock, aBlockchain);
    } else {
        return latestBlock.difficulty;
    }
};

const getAdjustedDifficulty = (latestBlock: Block, aBlockchain: Block[]) => {
    const prevAdjustmentBlock: Block = aBlockchain[blockchain.length - DIFFICULTY_ADJUSTMENT_INTERVAL];
    const timeExpected: number = BLOCK_GENERATION_INTERVAL * DIFFICULTY_ADJUSTMENT_INTERVAL;
    const timeTaken: number = latestBlock.timestamp - prevAdjustmentBlock.timestamp;
    if (timeTaken < timeExpected / 2) {
        return prevAdjustmentBlock.difficulty + 1;
    } else if (timeTaken > timeExpected * 2) {
        return prevAdjustmentBlock.difficulty - 1;
    } else {
        return prevAdjustmentBlock.difficulty;
    }
};
```

### Timestamp validation
The timestamp did not have any role nor validation. In fact it could be anything the client decided to generate. This changes now that the difficulty adjustment is introduced as the timeTaken variable (in the previous code snippet) is calculated based on the timestamps of the blocks.

To mitigate the attack where a false timestamp is introduced in order to manipulate the difficulty the following rules is introduced:
* A block is valid, if the timestamp is at most 1 min in the future from the time we perceive.
* A block in the chain is valid, if the timestamp is at most 1 min in the past of the previous block.
```
const isValidTimestamp = (newBlock: Block, previousBlock: Block): boolean => {
    return ( previousBlock.timestamp - 60 < newBlock.timestamp )
        && newBlock.timestamp - 60 < getCurrentTimestamp();
};
```

### Cumulative difficulty
We chose always the "longest" blockchain to be the valid. This must change now that difficulty is introduced. For now on the "correct" chain is not the "longest" chain, but the chain with the most cumulative difficulty. In other words, the correct chain is the chain which required most resources (= hashRate * time) to produce.

To get the cumulative difficulty of a chain we calculate 2^difficulty for each block and take a sum of all those numbers. We have to use the 2^difficulty as we chose the difficulty to represent the number of zeros that must prefix the hash in binary format. For instance, if we compare the difficulties of 5 and 11, it requires 2^(11-5) = 2^6 times more work to find a block with latter difficulty.

In the below example, the "Chain B" is the "correct" chain although it has fever blocks: 

![alt tag](https://i.imgur.com/AnciP6V.png)

Only the difficulty of the block matters, not the actual hash (given the hash is valid). For example, if the difficulty is 4 and the block hash is 000000a34c… (= also satisfying the difficulty of 6), only the difficulty of 4 is taken into account when calculating the cumulative difficulty.

This property is also known as "Nakamoto consensus" and it is one of the most important inventions Satoshi made, when s/he invented Bitcoin. In case of forks, miners must choose on which chain the they decide put their current resources (= hashRate). As it is in the interest of the miners to produce such block that will be included in the blockchain, the miners are incentivized to eventually to choose the same chain.

### Conclusions
An important property that a Proof-of-work puzzle must have is that it is difficult to solve, but easy to verify. Finding specific SHA256 hashes is a good and simple example of such problem. We implemented the difficulty aspect and nodes must now "mine" in order to add new blocks to the chain.

## 4) TRANSACTIONS
### Overview
We will introduce the concept of transactions. With this modification, we actually shift from our project from a "general purpose" blockchain to a cryptocurrency. As a result, we can send coins to addresses if we can show a proof that we own them in the first place.

To enable all this, a lot of new concepts must presented. This includes public-key cryptography, signatures and transactions inputs and outputs.

### Public-key cryptography and signatures
In Public-key cryptography you have a keypair: a secret key and a public key. The public key can be derived from the secret key, but the secret key cannot be derived from the public key. The public key (as the name implies) can be shared safely to anyone.

Any messages can be signed using the private key to create a signature. With this signature and the corresponding public key, anyone can verify that the signature is produced by the private key the that matches the public key.

![alt tag](https://i.imgur.com/jALm7T2.png)

We will use a library called elliptic for the public-key cryptography, which uses elliptic curves. (= ECDSA)

Conclusively, two different cryptographic functions are used for different purposes in the cryptocurrency:
* Hash function (SHA256) for the Proof-of-work mining (The hash is also used to preserve block integrity)
* Public-key cryptography (ECDSA) for transactions

### Private-keys and public keys (in ECDSA)
A valid private key is any random 32 byte string, eg. 19f128debc1b9122da0635954488b208b829879cf13b3d6cac5d1260c0fd967c

A valid public key is ‘04’ concatenated with a 64 byte string, e.g 04bfcab8722991ae774db48f934ca79cfb7dd991229153b9f732ba5334aafcd8e7266e47076996b55a14bf9913ee3145ce0cfc1372ada8ada74bd287450313534a

The public key can be derived from the private key. The public-key will be used as the ‘receiver’ (= address) of the coins in a transaction.

### Transactions overview
Before writing any code, let’s get an overview about the structure of transactions. Transactions consists of two components: inputs and outputs. Outputs specify where the coins are sent and inputs give a proof that the coins that are actually sent exists in the first place and are owned by the "sender". Inputs always refer to an existing (unspent) output.

![alt tag](https://i.imgur.com/GbgPqX0.png)

### Transaction outputs
Transaction outputs (txOut) consists of an address and an amount of coins. The address is an ECDSA public-key. This means that the user having the private-key of the referenced public-key (=address) will be able to access the coins.
```
class TxOut {
    public address: string;
    public amount: number;

    constructor(address: string, amount: number) {
        this.address = address;
        this.amount = amount;
    }
}
```

### Transaction inputs
Transaction inputs (txIn) provide the information "where" the coins are coming from. Each txIn refer to an earlier output, from which the coins are ‘unlocked’, with the signature. These unlocked coins are now ‘available’ for the txOuts. The signature gives proof that only the user, that has the private-key of the referred public-key ( =address) could have created the transaction.
```
class TxIn {
    public txOutId: string;
    public txOutIndex: number;
    public signature: string;
}
```

It should be noted that the txIn contains only the signature (created by the private-key), never the private-key itself. The blockchain contains public-keys and signatures, never private-keys.

As a conclusion, it can also be thought that the txIns unlock the coins and the txOuts ‘relock’ the coins: 

![alt tag](https://i.imgur.com/LU0g9vT.png)

### Transaction structure
The transactions structure itself is quite simple as we have now defined txIns and txOuts.

### Transaction id
The transaction id is calculated by taking a hash from the contents of the transaction. However, the signatures of the txIds are not included in the transaction hash as the will be added later on to the transaction.
```
const getTransactionId = (transaction: Transaction): string => {
    const txInContent: string = transaction.txIns
        .map((txIn: TxIn) => txIn.txOutId + txIn.txOutIndex)
        .reduce((a, b) => a + b, '');

    const txOutContent: string = transaction.txOuts
        .map((txOut: TxOut) => txOut.address + txOut.amount)
        .reduce((a, b) => a + b, '');

    return CryptoJS.SHA256(txInContent + txOutContent).toString();
};
```

### Transaction signatures
It is important that the contents of the transaction cannot be altered, after it has been signed. As the transactions are public, anyone can access to the transactions, even before they are included in the blockchain.

When signing the transaction inputs, only the txId will be signed. If any of the contents in the transactions is modified, the txId must change, making the transaction and signature invalid.
```
const signTxIn = (transaction: Transaction, txInIndex: number,
                  privateKey: string, aUnspentTxOuts: UnspentTxOut[]): string => {
    const txIn: TxIn = transaction.txIns[txInIndex];
    const dataToSign = transaction.id;
    const referencedUnspentTxOut: UnspentTxOut = findUnspentTxOut(txIn.txOutId, txIn.txOutIndex, aUnspentTxOuts);
    const referencedAddress = referencedUnspentTxOut.address;
    const key = ec.keyFromPrivate(privateKey, 'hex');
    const signature: string = toHexString(key.sign(dataToSign).toDER());
    return signature;
};
```

Let’s try to understand what happens if someone tries to modify the transaction:
1) Attacker runs a node and receives a transaction with content: "send 10 coins from address AAA to BBB" with txId 0x555..
2) The attacker changes the receiver address to CCC and relays it forward in the network. Now the content of the transaction is "send 10 coins from address AAA to CCC"
3) However, as the receiver address is changed, the txId is not valid anymore. A new valid txId would be 0x567...
4) If the txId is set to the new value, the signature is not valid. The signature matches only with the original txId 0x555..
5) The modified transaction will not be accepted by other nodes, since either way, it is invalid.

### Unspent transaction outputs
A transaction input must always refer to an unspent transaction output (uTxO). Consequently, when you own some coins in the blockchain, what you actually have is a list of unspent transaction outputs whose public key matches to the private key you own.

In terms of transactions validation, we can only focus on the list of unspent transactions outputs, in order to figure out if the transaction is valid. The list of unspent transaction outputs can always be derived from the current blockchain. In this implementation, we will update the list of unspent transaction outputs as we process and include the transactions to the blockchain.

The data structure for an unspent transaction output looks like this:
```
class UnspentTxOut {
    public readonly txOutId: string;
    public readonly txOutIndex: number;
    public readonly address: string;
    public readonly amount: number;

    constructor(txOutId: string, txOutIndex: number, address: string, amount: number) {
        this.txOutId = txOutId;
        this.txOutIndex = txOutIndex;
        this.address = address;
        this.amount = amount;
    }
}
```

The data structure itself if just a list:
```
let unspentTxOuts: UnspentTxOut[] = [];
```

### Updating unspent transaction outputs
Every time a new block is added to the chain, we must update our list of unspent transaction outputs. This is because the new transactions will spend some of the existing transaction outputs and introduce new unspent outputs.

To handle this, we will first retrieve all new unspent transaction outputs (newUnspentTxOuts) from the new block:
```
const newUnspentTxOuts: UnspentTxOut[] = newTransactions
    .map((t) => {
        return t.txOuts.map((txOut, index) => new UnspentTxOut(t.id, index, txOut.address, txOut.amount));
        })
    .reduce((a, b) => a.concat(b), []);
```

We will also need to know which transaction outputs are consumed by the new transactions of the block (consumedTxOuts). This will be solved by examining the inputs of the new transactions:
```
const consumedTxOuts: UnspentTxOut[] = newTransactions
    .map((t) => t.txIns)
    .reduce((a, b) => a.concat(b), [])
    .map((txIn) => new UnspentTxOut(txIn.txOutId, txIn.txOutIndex, '', 0));
```

Finally, we can generate the new unspent transaction outputs by removing the consumedTxOuts and adding the newUnspentTxOuts to our existing transaction outputs.
```
const resultingUnspentTxOuts = aUnspentTxOuts
    .filter(((uTxO) => !findUnspentTxOut(uTxO.txOutId, uTxO.txOutIndex, consumedTxOuts)))
    .concat(newUnspentTxOuts);
```

The described code and functionality is contained in the updateUnspentTxOuts method. It should be noted that this method is called only after the transactions in the block (and the block itself) has been validated.

### Transactions validation
We can now finally lay out the rules what makes a transaction valid:

### Correct transaction structure
The transaction must conform with the defined classes of Transaction, TxIn and TxOut
```
const isValidTransactionStructure = (transaction: Transaction) => {
    if (typeof transaction.id !== 'string') {
        console.log('transactionId missing');
        return false;
		}
        ...
       //check also the other members of class
    }
```

### Valid transaction id
The id in the transaction must be correctly calculated.
```
if (getTransactionId(transaction) !== transaction.id) {
        console.log('invalid tx id: ' + transaction.id);
        return false;
    }
```

### Valid txIns
The signatures in the txIns must be valid and the referenced outputs must have not been spent.
```
const validateTxIn = (txIn: TxIn, transaction: Transaction, aUnspentTxOuts: UnspentTxOut[]): boolean => {
    const referencedUTxOut: UnspentTxOut =
        aUnspentTxOuts.find((uTxO) => uTxO.txOutId === txIn.txOutId && uTxO.txOutId === txIn.txOutId);
    if (referencedUTxOut == null) {
        console.log('referenced txOut not found: ' + JSON.stringify(txIn));
        return false;
    }
    const address = referencedUTxOut.address;

    const key = ec.keyFromPublic(address, 'hex');
    return key.verify(transaction.id, txIn.signature);
};
```

### Valid txOut values
The sums of the values specified in the outputs must be equal to the sums of the values specified in the inputs. If you refer to an output that contains 50 coins, the sum of the values in the new outputs must also be 50 coins.
```
const totalTxInValues: number = transaction.txIns
    .map((txIn) => getTxInAmount(txIn, aUnspentTxOuts))
    .reduce((a, b) => (a + b), 0);
const totalTxOutValues: number = transaction.txOuts
    .map((txOut) => txOut.amount)
    .reduce((a, b) => (a + b), 0);

if (totalTxOutValues !== totalTxInValues) {
    console.log('totalTxOutValues !== totalTxInValues in tx: ' + transaction.id);
    return false;
    }
```

### Coinbase transaction
Transaction inputs must always refer to unspent transaction outputs, but from where does the initial coins come in to the blockchain? To solve this, a special type of transaction is introduced: coinbase transaction

The coinbase transaction contains only an output, but no inputs. This means that a coinbase transaction adds new coins to circulation. We specify the amount of the coinbase output to be 50 coins.
```
const COINBASE_AMOUNT: number = 50;
```

The coinbase transaction is always the first transaction in the block and it is included by the miner of the block. The coinbase reward acts as an incentive for the miners: if you find the block, you are able to collect 50 coins.

We will add the block height to input of the coinbase transaction. This is to ensure that each coinbase transaction has a unique txId. Without this rule, for instance, a coinbase transaction stating "give 50 coins to address 0xabc" would always have the same txId.

The validation of the coinbase transaction differs slightly from the validation of a "normal" transaction
```
const validateCoinbaseTx = (transaction: Transaction, blockIndex: number): boolean => {
    if (getTransactionId(transaction) !== transaction.id) {
        console.log('invalid coinbase tx id: ' + transaction.id);
        return false;
    }
    if (transaction.txIns.length !== 1) {
        console.log('one txIn must be specified in the coinbase transaction');
        return;
    }
    if (transaction.txIns[0].txOutIndex !== blockIndex) {
        console.log('the txIn index in coinbase tx must be the block height');
        return false;
    }
    if (transaction.txOuts.length !== 1) {
        console.log('invalid number of txOuts in coinbase transaction');
        return false;
    }
    if (transaction.txOuts[0].amount != COINBASE_AMOUNT) {
        console.log('invalid coinbase amount in coinbase transaction');
        return false;
    }
    return true;
};
```

### Conclusions
We included the concept of transactions to the blockchain. The basic idea is quite simple: we refer to unspent outputs in transaction inputs and use signatures to show that the unlocking part is valid. We then use outputs to "relock" them to a receiver address.

However, creating transactions is still very difficult. We must manually create the inputs and outputs of the transactions and sign them using our private keys.

There is also no transaction relaying yet: to include a transaction to the blockchain, you must mine it yourself. This is also the reason we did not yet introduce the concept of transaction fee.

## 5) WALLET
### Overview
The goal of the wallet is to create a more abstract interface for the end user.

The end user must be able to
* Create a new wallet (=private key in this case)
* View the balance of his wallet
* Send coins to other addresses

All of the above must work so that the end user must not need to understand how txIns or txOuts work. Just like in e.g. Bitcoin: you send coins to addresses and publish your own address where other people can send coins.

### Generating and storing the private key
In this tutorial we will use the simplest possible way to handle the wallet generation and storing: We will generate an unencrypted private key to the file node/wallet/private_key.
```
const privateKeyLocation = 'node/wallet/private_key';

const generatePrivatekey = (): string => {
    const keyPair = EC.genKeyPair();
    const privateKey = keyPair.getPrivate();
    return privateKey.toString(16);
};

const initWallet = () => {
    //let's not override existing private keys
    if (existsSync(privateKeyLocation)) {
        return;
    }
    const newPrivateKey = generatePrivatekey();

    writeFileSync(privateKeyLocation, newPrivateKey);
    console.log('new wallet with private key created');
};
```

And as said, the public key (=address) can be calculated from the private key.
```
const getPublicFromWallet = (): string => {
    const privateKey = getPrivateFromWallet();
    const key = EC.keyFromPrivate(privateKey, 'hex');
    return key.getPublic().encode('hex');
};
```

It should be noted that storing the private key in an unencrypted format is very unsafe. We do this only for the purpose to keep things simple for now. Also, the wallet supports only a single private key, so you need to generate a new wallet to get a new public address.

### Wallet balance
When you own some coins in the blockchain, what you actually have is a list of unspent transaction outputs whose public key matches to the private key you own.

This means that calculating the balance for a given address is quite simple: you just sum all the unspent transaction "owned" by that address:
```
const getBalance = (address: string, unspentTxOuts: UnspentTxOut[]): number => {
    return _(unspentTxOuts)
        .filter((uTxO: UnspentTxOut) => uTxO.address === address)
        .map((uTxO: UnspentTxOut) => uTxO.amount)
        .sum();
};
```

As demonstrated in the code, the private key is not needed to query the balance of the address. This consequently means that anyone can solve the balance of a given address.

### Generating transactions
When sending coins, the user should be able to ignore the concepts of transaction inputs and outputs. But what should happen if the user A has balance of 50 coins, that is in a single transaction output and the user wants to send 10 coins to user B?

In this case, the solution is to send 10 bitcoins to the address of user B and 40 coins back to user A. The full transaction output must always be spent, so the "splitting" part must be done when assigning the coins to new outputs. This simple case is demonstrated in the following out picture (txIns are not shown):

![alt tag](https://i.imgur.com/i1Y0eXB.png)

Let’s play out a bit more complex transaction scenario:
1) User C has initially 0 coins
2) User C receives 3 transactions worth of 10, 20 and 30 coins
3) User C wants to send 55 coins to user D. What will the transaction look like?

In this case, all of the three outputs must be used and the outputs must have values of 55 coins to user D and 5 coins back to user C.

![alt tag](https://i.imgur.com/dEI1qyO.png)

Let’s manifest the described logic to code. First we will create the transaction inputs. To do this, we will loop through our unspent transaction outputs until the sum of these outputs is greater or equal than the amount we want to send.
```
const findTxOutsForAmount = (amount: number, myUnspentTxOuts: UnspentTxOut[]) => {
    let currentAmount = 0;
    const includedUnspentTxOuts = [];
    for (const myUnspentTxOut of myUnspentTxOuts) {
        includedUnspentTxOuts.push(myUnspentTxOut);
        currentAmount = currentAmount + myUnspentTxOut.amount;
        if (currentAmount >= amount) {
            const leftOverAmount = currentAmount - amount;
            return {includedUnspentTxOuts, leftOverAmount}
        }
    }
    throw Error('not enough coins to send transaction');
};
```

As shown, we will also calculate the leftOverAmount which is the value we will send back to our address.

As we have the list of unspent transaction outputs, we can create the txIns of the transaction:
```
const toUnsignedTxIn = (unspentTxOut: UnspentTxOut) => {
    const txIn: TxIn = new TxIn();
    txIn.txOutId = unspentTxOut.txOutId;
    txIn.txOutIndex = unspentTxOut.txOutIndex;
    return txIn;
};
const {includedUnspentTxOuts, leftOverAmount} = findTxOutsForAmount(amount, myUnspentTxouts);
const unsignedTxIns: TxIn[] = includedUnspentTxOuts.map(toUnsignedTxIn);
```

Next the two txOuts of the transaction are created: One txOut for the receiver of the coins and one txOut for the leftOverAmount`. If the txIns happen to have the exact amount of the desired value (leftOverAmount is 0) we do not create the "leftOver" transaction.
```
const createTxOuts = (receiverAddress:string, myAddress:string, amount, leftOverAmount: number) => {
    const txOut1: TxOut = new TxOut(receiverAddress, amount);
    if (leftOverAmount === 0) {
        return [txOut1]
    } else {
        const leftOverTx = new TxOut(myAddress, leftOverAmount);
        return [txOut1, leftOverTx];
    }
};
```

Finally we calculate the transaction id and sign the txIns:
```
const tx: Transaction = new Transaction();
    tx.txIns = unsignedTxIns;
    tx.txOuts = createTxOuts(receiverAddress, myAddress, amount, leftOverAmount);
    tx.id = getTransactionId(tx);

    tx.txIns = tx.txIns.map((txIn: TxIn, index: number) => {
        txIn.signature = signTxIn(tx, index, privateKey, unspentTxOuts);
        return txIn;
    });
```

### Using the wallet
Let’s also add a meaningful controlling endpoint to for the wallet functionality.
```
app.post('/mineTransaction', (req, res) => {
    const address = req.body.address;
    const amount = req.body.amount;
    const resp = generatenextBlockWithTransaction(address, amount);
    res.send(resp);
    });
```

As it is shown, the end user must only provide the address and the amount of coins for the node. The node will calculate the rest.

### Conclusions
We just implemented a Ted unencrypted wallet with simple transaction generation. Although this transaction generation algorithm never creates transactions with more than 2 outputs, it should be noted that the blockchain itself supports any number of outputs. You could create valid a transaction with input of 50 coins and output of 5,15 and 30 coins, but those must be created manually using the /mineRawBlock interface.

Also, the only way to include a desired transaction in the blockchain is to mine it yourself. The nodes do not exchange information about transactions that are not yet included in the blockchain.

## 6) TRANSACTION RELAYING
### Overview
We will implement the relaying of such transactions, that are not yet included in the blockchain. In bitcoin, these transaction are also known as "unconfirmed transactions". Typically, when someone wants to include a transaction to the blockchain (= send coins to some address ) he broadcasts the transaction to the network and hopefully some node will mine the transaction to the blockchain.

This feature is very important for a working cryptocurrency, since it means you don’t need to mine a block yourself, in order to include a transaction to the blockchain.

As a consequence, the nodes will now share two types of data when they communicate with each other:
* The state of the blockchain ( =the blocks and transactions that are included to the blockchain)
* Unconfirmed transactions ( =the transactions that are not yet included in the blockchain)

### Transaction pool
We will store our unconfirmed transactions in a new entity called "transaction pool" (also known as "mempool" in bitcoin). Transaction pool is a structure that contains all of the "unconfirmed transactions" our node know of. In this simple implementation we will just use a list.
```
let transactionPool: Transaction[] = [];
```

We will also introduce a new endpoint to our node: POST /sendTransaction. This method creates the a transaction to our local transaction pool based on the existing wallet functionality. For now on we will use this method as the "preferred" interface when we want to include a new transaction to the blockchain.
```
app.post('/sendTransaction', (req, res) => {
        ...
    })
```

We create the transaction. We just add the created transaction to the pool instead of instantly trying to mine a block:
```
const sendTransaction = (address: string, amount: number): Transaction => {
    const tx: Transaction = createTransaction(address, amount, getPrivateFromWallet(), getUnspentTxOuts(), getTransactionPool());
    addToTransactionPool(tx, getUnspentTxOuts());
    return tx;
};
```

### Broadcasting
The whole point of the unconfirmed transactions are that they will spread throughout the network and eventually some node will mine the transaction to the blockchain. To handle this we will introduce the following simple rules for the networking of unconfirmed transactions:
* When a node receives an unconfirmed transaction it has not seen before, it will broadcast the full transaction pool to all peers.
* When a node first connects to another node, it will query for the transaction pool of that node.

We will add two new MessageTypes to serve this purpose: QUERY_TRANSACTION_POOL and RESPONSE_TRANSACTION_POOL. The MessageType enum will now look now like this:
```
enum MessageType {
    QUERY_LATEST = 0,
    QUERY_ALL = 1,
    RESPONSE_BLOCKCHAIN = 2,
    QUERY_TRANSACTION_POOL = 3,
    RESPONSE_TRANSACTION_POOL = 4
}
```

The transaction pool messages will be created in the following way:
```
const responseTransactionPoolMsg = (): Message => ({
    'type': MessageType.RESPONSE_TRANSACTION_POOL,
    'data': JSON.stringify(getTransactionPool())
}); 

const queryTransactionPoolMsg = (): Message => ({
    'type': MessageType.QUERY_TRANSACTION_POOL,
    'data': null
});
```

To implement the described transaction broadcasting logic, we add code to handle the MessageType.RESPONSE_TRANSACTION_POOL message type. Whenever, we receive unconfirmed transactions, we try to add those to our transaction pool. If we manage to add a transaction to our pool, it means that the transaction is valid and our node has not seen the transaction before. In this case we broadcast our own transaction pool to all peers.
```
case MessageType.RESPONSE_TRANSACTION_POOL:
    const receivedTransactions: Transaction[] = JSONToObject<Transaction[]>(message.data);
    receivedTransactions.forEach((transaction: Transaction) => {
        try {
            handleReceivedTransaction(transaction);
            //if no error is thrown, transaction was indeed added to the pool
            //let's broadcast transaction pool
            broadCastTransactionPool();
        } catch (e) {
            //unconfirmed transaction not valid (we probably already have it in our pool)
        }
    });
```

***Validating received unconfirmed transactions
As the peers can send us any kind of transactions, we must validate the transactions before we can add them to the transaction pool. All of the existing transaction validation rules apply. For instance, the transaction must be correctly formatted, and the transaction inputs, outputs and signatures must match.

In addition to the existing rules, we add a new rule: a transaction cannot be added to the pool if any of the transaction inputs are already found in the existing transaction pool. This new rule is embodied in the following code:
```
const isValidTxForPool = (tx: Transaction, aTtransactionPool: Transaction[]): boolean => {
    const txPoolIns: TxIn[] = getTxPoolIns(aTtransactionPool);

    const containsTxIn = (txIns: TxIn[], txIn: TxIn) => {
        return _.find(txPoolIns, (txPoolIn => {
            return txIn.txOutIndex === txPoolIn.txOutIndex && txIn.txOutId === txPoolIn.txOutId;
        }))
    };

    for (const txIn of tx.txIns) {
        if (containsTxIn(txPoolIns, txIn)) {
            console.log('txIn already found in the txPool');
            return false;
        }
    }
    return true;
};
```

There is no explicit way to remove a transaction from the transaction pool. The transaction pool will however be updated each time a new block is found.

### From transaction pool to blockchain
Let’s next implement a way for the unconfirmed transaction to find its way from the local transaction pool to a block mined by the same node. This is simple: when a node starts to mine a block, it will include the transactions from the transaction pool to the new block candidate.
```
const generateNextBlock = () => {
    const coinbaseTx: Transaction = getCoinbaseTransaction(getPublicFromWallet(), getLatestBlock().index + 1);
    const blockData: Transaction[] = [coinbaseTx].concat(getTransactionPool());
    return generateRawNextBlock(blockData);
};
```

As the transactions are already validated, before they are added to the pool, we are not doing any further validations at this points.

### Updating the transaction pool
As new blocks with transactions are mined to the blockchain, we must revalidate the transaction pool every time a new block is found. It is possible that the new block contains transactions that makes some of the transactions in the pool invalid. This can happen if for instance:
* The transaction that was in the pool was mined (by the node itself or by someone else)
* The unspent transaction output that is referred in the unconfirmed transaction is spent by some other transaction

The transaction pool will be updated with the following code:
```
const updateTransactionPool = (unspentTxOuts: UnspentTxOut[]) => {
    const invalidTxs = [];
    for (const tx of transactionPool) {
        for (const txIn of tx.txIns) {
            if (!hasTxIn(txIn, unspentTxOuts)) {
                invalidTxs.push(tx);
                break;
            }
        }
    }
    if (invalidTxs.length > 0) {
        console.log('removing the following transactions from txPool: %s', JSON.stringify(invalidTxs));
        transactionPool = _.without(transactionPool, ...invalidTxs)
    }
};
```

As it can be seen, we need to know only the current unspent transaction outputs to make the decision if a transaction should be removed from the pool.

### Conclusions
We can now include transactions to the blockchain without actually having to mine the blocks themselves. There is however no incentive for the nodes to include a received transaction to the block as we did not implement the concept of transaction fees.

## 7) WALLET UI AND BLOCKCHAIN EXPLORER
### Overview
We will add an UI for the wallet and create blockchain explorer for our blockchain. Our node already exposes its functionalities with HTTP endpoints, so we will create a web page that makes requests to those endpoints and visualizes the results.

To achieve all this, we must add some additional endpoints and logic your node, for instance:
* Query information about blocks and transactions
* Query information about a specific address

### New endpoints
Let’s add an endpoint from which the user can query a specific block, if the hash is known.
```
app.get('/block/:hash', (req, res) => {
    const block = _.find(getBlockchain(), {'hash' : req.params.hash});
    res.send(block);
    }); 
```

The same goes for querying a specific transaction:
```
app.get('/transaction/:id', (req, res) => {
    const tx = _(getBlockchain())
        .map((blocks) => blocks.data)
        .flatten()
        .find({'id': req.params.id});
    res.send(tx);
    });
```

We would also like to show information about a specific address. For now we return the list of unspent outputs for that address, as from this information we can e.g. calculate the total balance for that address.
```
app.get('/address/:address', (req, res) => {
    const unspentTxOuts: UnspentTxOut[] =
		_.filter(getUnspentTxOuts(), (uTxO) => uTxO.address === req.params.address)
    res.send({'unspentTxOuts': unspentTxOuts});
    });
```

We could also add information about spent transaction outputs for a given address in order to visualize full history of a given address.

### Frontend techs
We will use Vue.js to implement the UI parts of the wallet and the blockchain explorer. Since this tutorial is not about frontend development, we will not walk through the frontent part in terms of code. 

### Blockchain explorer
Blockchain explorer is a website which is used to visualize the state of the blockchain. A typical use case for a blockchain explorer is to easily check the balance of a given address or verify that a given transaction is included to the blockchain.

In our case we simply make a http requests to the node and show the responses in a some meaningful way. We never make any request that modifys the state of the blockchain, so building a blockchain explorer is all about visualizing the information the node provides in a meaningful way.

A screenshot from the blockchain explorer:

![alt tag](https://i.imgur.com/wJwIKBI.png)

### Wallet UI
For the wallet UI we will also create a similar website as in the case of blockchain explorer. The user should be able to send coins and view the balance of address. We will also show the transaction pool.

A screenshot from the wallet:

![alt tag](https://i.imgur.com/m5G3I58.png)