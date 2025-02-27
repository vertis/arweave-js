# Arweave JS

Arweave JS is the JavaScript/TypeScript SDK for interacting with the Arweave network and uploading data ot the permaweb. It works in latest browsers and Node JS.

- [Arweave JS](#arweave-js)
  - [Installation](#installation)
    - [NPM](#npm)
    - [Bundles](#bundles)
  - [Initialisation](#initialisation)
    - [NPM Node](#npm-node)
    - [NPM Web](#npm-web)
    - [Web Bundles](#web-bundles)
    - [Initialisation options](#initialisation-options)
  - [Usage](#usage)
    - [Wallets and Keys](#wallets-and-keys)
      - [Create a new wallet and private key](#create-a-new-wallet-and-private-key)
      - [Get the wallet address for a private key](#get-the-wallet-address-for-a-private-key)
      - [Get a wallet balance](#get-a-wallet-balance)
      - [Get the last transaction ID from a wallet](#get-the-last-transaction-id-from-a-wallet)
    - [Transactions](#transactions)
      - [Create a data transaction](#create-a-data-transaction)
      - [Create a wallet to wallet transaction](#create-a-wallet-to-wallet-transaction)
      - [Add tags to a transaction](#add-tags-to-a-transaction)
      - [Sign a transaction](#sign-a-transaction)
      - [Submit a transaction](#submit-a-transaction)
      - [Get a transaction status](#get-a-transaction-status)
      - [Get a transaction](#get-a-transaction)
      - [Decode data and tags from transactions](#decode-data-and-tags-from-transactions)
    - [ArQL](#arql)

## Installation
### NPM
```bash
npm install --save arweave
```

### Bundles
Single bundle file (web only - use the NPM method if using Node).

```html
<!-- Latest -->
<script src="https://unpkg.com/arweave/bundles/web.bundle.js"></script>

<!-- Latest, minified-->
<script src="https://unpkg.com/arweave/bundles/web.bundle.min.js"></script>

<!-- Specific version -->
<script src="https://unpkg.com/arweave@1.2.0/bundles/web.bundle.js"></script>

<!-- Specific version, minified -->
<script src="https://unpkg.com/arweave@1.2.0/bundles/web.bundle.min.js"></script>
```


## Initialisation

### NPM Node
```js
const Arweave = require('arweave/node');

const instance = Arweave.init({
    host: '127.0.0.1',
    port: 1984
});
```

### NPM Web
```js
import Arweave from 'arweave/web';

const arweave = Arweave.init({
    host: '127.0.0.1',
    port: 1984
});
```

### Web Bundles
```js
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Hello world</title>
    <script src="https://unpkg.com/arweave/bundles/web.bundle.js"></script>
    <script>
    const arweave = Arweave.init({
        host: '127.0.0.1',
        port: 1984
    });
    arweave.network.getInfo().then(console.log);
    </script>
</head>
<body>
    
</body>
</html>
```

The default port for nodes is `1984`.

A live list of public arweave nodes and IP adddresses can be found on this [peer explorer](http://arweave.net/bNbA3TEQVL60xlgCcqdz4ZPHFZ711cZ3hmkpGttDt_U).

### Initialisation options
```js
{
    host: 'arweave.net',// Hostname or IP address for a Arweave node
    port: 443,           // Port, defaults to 1984
    protocol: 'https',  // Network protocol http or https, defaults to http
    timeout: 20000,     // Network request timeouts in milliseconds
    logging: false,     // Enable network request logging
}
```

## Usage

### Wallets and Keys

#### Create a new wallet and private key

Here you can generate a new wallet address and private key ([JWK](https://docs.arweave.org/developers/server/http-api#key-format)), don't expose private keys or make them public as anyone with the key can use the corresponding wallet.

Make sure they're stored securely as they can never be recovered if lost.

Once AR has been sent to the address for a new wallet, the key can then be used to sign outgoing transactions.
```js
arweave.wallets.generate().then((key) => {
    console.log(key);
    // {
    //     "kty": "RSA",
    //     "n": "3WquzP5IVTIsv3XYJjfw5L-t4X34WoWHwOuxb9V8w...",
    //     "e": ...

    arweave.wallets.jwkToAddress(jwk).then((address) => {
        console.log(address);
        // 1seRanklLU_1VTGkEk7P0xAwMJfA7owA1JHW5KyZKlY
    )};
});
```

#### Get the wallet address for a private key

```js
arweave.wallets.jwkToAddress(jwk).then((address) => {
    console.log(address);
    //1seRanklLU_1VTGkEk7P0xAwMJfA7owA1JHW5KyZKlY
)};
```

#### Get an address balance
Get the balance of a wallet address, all amounts by default are returned in [winston](https://docs.arweave.org/developers/server/http-api#ar-and-winston).
```js
arweave.wallets.getBalance('1seRanklLU_1VTGkEk7P0xAwMJfA7owA1JHW5KyZKlY').then((balance) => {
    let winston = balance;
    let ar = arweave.ar.winstonToAr(balance);

    console.log(winston);
    //125213858712

    console.log(ar);
    //0.125213858712
});
```

#### Get the last transaction ID from a wallet

```js
arweave.wallets.getLastTransactionID('1seRanklLU_1VTGkEk7P0xAwMJfA7owA1JHW5KyZKlY').then((transactionId) => {
    console.log(transactionId);
    //3pXpj43Tk8QzDAoERjHE3ED7oEKLKephjnVakvkiHF8
});
```

### Transactions

Transactions are the building blocks of the Arweave permaweb, they can send [AR](https://docs.arweave.org/developers/server/http-api#ar-and-winston) betwen wallet addresses, or store data on the Arweave network.

The create transaction methods simply creates and returns an unsigned transaction object, you must sign the transaction and submit it separeately using the `transactions.sign` and `transactions.submit` methods.

**Modifying a transaction object after signing it will invalidate the signature**, this will cause it to be rejected by the network if submitted in that state. Transaction prices are based on the size of the data field, so modifying the data field after a transaction has been created isn't recommended as you'll need to manually update the price.

The transaction ID is a hash of the transaction signature, so a transaction ID can't be known until its contents are finalised and it has been signed.

#### Create a data transaction

Data transactions are used to store data on the Arweave permaweb, they can contain HTML data and be serverd like webpages or they can contain any arbitrary data.

```js
let key = await arweave.wallets.generate();

// Plain text
let transactionA = arweave.createTransaction({
    data: '<html><head><meta charset="UTF-8"><title>Hello world!</title></head><body></body></html>'
}, jwk);

// Buffer
let transactionB = arweave.createTransaction({
    data: Buffer.from('Some data', 'utf8')
}, jwk);


console.log(transactionA);
// Transaction {
//   last_tx: '',
//   owner: 'wgfbaaSXJ8dszMabPo-...',
//   tags: [],
//   target: '',
//   quantity: '0',
//   data: 'eyJhIjoxfQ',
//   reward: '321879995',
//   signature: '' }
```

#### Create a wallet to wallet transaction

```js
let key = await arweave.wallets.generate();

// Send 10.5 AR to 1seRanklLU_1VTGkEk7P0xAwMJfA7owA1JHW5KyZKlY
let transaction = arweave.createTransaction({
    target: '1seRanklLU_1VTGkEk7P0xAwMJfA7owA1JHW5KyZKlY',
    quantity: arweave.ar.arToWinston('10.5')
}, jwk);

console.log(transaction);
// Transaction {
//   last_tx: '',
//   owner: '14fXfoRDMFS5yTpUT7ODzj...',
//   tags: [],
//   target: '1seRanklLU_1VTGkEk7P0xAwMJfA7owA1JHW5KyZKlY',
//   quantity: '10500000000000',
//   data: '',
//   reward: '2503211
//   signature: '' }
```

#### Add tags to a transaction

Metadata can be added to transactions through tags, these are simple key/value attributes that can be used to document the contents of a transaction or provide related data.

ARQL uses tags when searching for transactions.

The `Content-Type` is a reserved tag and is used to set the data content type. For example, a transaction with HTML data and a content type tag of `text/html` will be served as a HTML page and render correctly in browsers,
if the content type is set to `text/plain` then it will be served as a plain text document and not render in browsers.

```js
let key = await arweave.wallets.generate();

let transaction = await arweave.createTransaction({
    data: '<html><head><meta charset="UTF-8"><title>Hello world!</title></head><body></body></html>',
}, key);

transaction.addTag('Content-Type', 'text/html');
transaction.addTag('key2', 'value2');

console.log(transaction);
// Transaction {
//   last_tx: '',
//   owner: 's8zPWNlBMiJFLcvpH98QxnI6FoPar3vCK3RdT...',
//   tags: [
//       Tag { name: 'Q29udGVudC1UeXBl', value: 'dGV4dC9odG1s' },
//       Tag { name: 'a2V5Mg', value: 'dmFsdWUy' }
//   ],
//   target: '',
//   quantity: '0',
//   data: 'PGh0bWw-PGhlYWQ-PG1ldGEgY2hh...',
//   reward: '329989175',
//   signature: '' }
```

#### Sign a transaction

```js
let key = await arweave.wallets.generate();

let transaction = await arweave.createTransaction({
    target: '1seRanklLU_1VTGkEk7P0xAwMJfA7owA1JHW5KyZKlY',
    quantity: arweave.ar.arToWinston('10.5')
}, key);

await arweave.transactions.sign(transaction, key);

console.log(transaction);
// Signature and id fields are now populated
// Transaction {
//   last_tx: '',
//   owner: '2xu89EaA5zENRRsbOh4OscMcy...',
//   tags: [],
//   target: '1seRanklLU_1VTGkEk7P0xAwMJfA7owA1JHW5KyZKlY',
//   quantity: '10500000000000',
//   data: '',
//   reward: '250321179212',
//   signature: 'AbFjlpEHTN6_SKWsUSMAzalImOVxNm86Z8hoTZcItkYBJLx...'
//   id: 'iHVHijWvKbIa0ZA9IbuKtOxJdNO9qyey6CIH324zQWI' 
```

#### Submit a transaction

Once a transaction is submitted to the network it'll be broadcast around all nodes and mined into a block.

```js
let key = await arweave.wallets.generate();

let transaction = await arweave.createTransaction({
    target: '1seRanklLU_1VTGkEk7P0xAwMJfA7owA1JHW5KyZKlY',
    quantity: arweave.ar.arToWinston('10.5')
}, key);

await arweave.transactions.sign(transaction, key);

const response = await arweave.transactions.post(transaction);

console.log(response.status);
// 200

// HTTP response codes (200 - ok, 400 - invalid transaction, 500 - error)
```

#### Get a transaction status

```js
arweave.transactions.getStatus('bNbA3TEQVL60xlgCcqdz4ZPHFZ711cZ3hmkpGttDt_U').then(status => {
    console.log(status);
    // 200
})
```

#### Get a transaction

Fetch a transaction from the connected arweave node. The data and tags are base64 encoded, these can be decoded using the built in helper methods.

```js
const transaction = arweave.transactions.get('bNbA3TEQVL60xlgCcqdz4ZPHFZ711cZ3hmkpGttDt_U').then(transaction => {
  console.log(transaction);
  // Transaction {
  //   last_tx: 'cO5gl_d5ARnaoBtu2Vas8skgLg-6KnC9gH8duWP7Ll8',
  //   owner: 'pJjRtSRLpHUVAKCtWC9pjajI_VEpiPEEAHX0k...',
  //   tags: [
  //       Tag { name: 'Q29udGVudC1UeXBl', value: 'dGV4dC9odG1s' },
  //       Tag { name: 'VXNlci1BZ2VudA', value: 'QXJ3ZWF2ZURlcGxveS8xLjEuMA' }
  //   ],
  //   target: '',
  //   quantity: '0',
  //   data: 'CjwhRE9DVFlQRSBodG1sPgo8aHRtbCBsYW5nPSJlbiI...',
  //   reward: '1577006493',
  //   signature: 'NLiRQSci56KVNk-x86eLT1TyF1ST8pzE...',
  //   id: 'bNbA3TEQVL60xlgCcqdz4ZPHFZ711cZ3hmkpGttDt_U' }
  // })
});
```

#### Decode data and tags from transactions

```js
const transaction = arweave.transactions.get('bNbA3TEQVL60xlgCcqdz4ZPHFZ711cZ3hmkpGttDt_U').then(transaction => {

  // Use the get method to get a specific transaction field.
  console.log(transaction.get('signature'));
  // NLiRQSci56KVNk-x86eLT1TyF1ST8pzE-s7jdCJbW-V...

  console.log(transaction.get('data'));
  //CjwhRE9DVFlQRSBodG1sPgo8aHRtbCBsYW5nPSJlbiI-C...

  // Get the data base64 decoded as a Uint8Array byte array.
  console.log(transaction.get('data', {decode: true}));
  //Uint8Array[10,60,33,68,79,67,84,89,80,69...

  // Get the data base64 decoded as a string.
  console.log(transaction.get('data', {decode: true, string: true}));
  //<!DOCTYPE html>
  //<html lang="en">
  //<head>
  //    <meta charset="UTF-8">
  //    <meta name="viewport" content="width=device-width, initial-scale=1.0">
  //    <title>ARWEAVE / PEER EXPLORER</title>

  transaction.get('tags').forEach(tag => {
    let key = tag.get('name', {decode: true, string: true});
    let value = tag.get('value', {decode: true, string: true});
    console.log(`${key} : ${value}`);
  });
  // Content-Type : text/html
  // User-Agent : ArweaveDeploy/1.1.0
});
```

### ArQL

#### Get a list of transaction IDs matching the giver query

ArQL allows you to search for transactions by tags or by wallet. Searching by wallet is done by using the special tag `from`. The allowed operators are `and`, `or`, and `equals` which all accept exactly two expressions. Therefore, to `and` three or more expressions together, you will need to nest `and` expressions. The same goes for `or`.

`arweave.arql` takes the ArQL query as a JavaScript object and returns the matching transaction IDs as an array or strings.

```javascript
const txids = arweave.arql({
  op: "and",
  expr1: {
    op: "equals",
    expr1: "from",
    expr2: "hnRI7JoN2vpv__w90o4MC_ybE9fse6SUemwQeY8hFxM"
  },
  expr2: {
    op: "or",
    expr1: {
      op: "equals",
      expr1: "type",
      expr2: "post"
    },
    expr2: {
      op: "equals",
      expr1: "type",
      expr2: "comment"
    }
  }
})

console.log(txids)
// [
//   'TwS2G8mi5JGypMZO_EWtHKvrJkB76hXmWN3ROCjkLBc',
//   'urdjQI4iKo7l8xQ0A55G7bOM3oi4QdGAd7MeVE_ru5c',
//   '_CD8p7z3uFJCB03OCMU7R80FTQ3ZRf8O2UGhNxoUaOg',
//   ...
// ]
```
