---
id: python-sdk
title: Kin SDK for Python
---

The Kin SDK for Python is meant to be used as a back-end service. It can perform some actions for your client apps (iOS, Android, etc.) and also operate as a server for you to build services on top of the Kin Blockchain. The SDK can for example take care of communicating with the Kin Blockchain on behalf of the client to create accounts and whitelist transactions and it can also monitor blockchain transactions so that you can implement broader services. It's up to you how to integrate the SDK in your overall architecture and manage server up-time.

## Requirements.

Make sure you have Python 3 >= 3.4

## Installation

```bash
pip install kin-sdk
```

Track the development of this SDK on [GitHub](https://github.com/kinecosystem/kin-sdk-python/tree/v2-master).

## Overview

In this introduction we will look at a few basic operations on the Kin Blockchain and some features that are exclusive to the Kin SDK for Python.

You will find:

* Accessing the Kin Blockchain
* Managing Kin accounts
* Executing transactions against Kin accounts
* Monitoring Kin Payments (Python SDK can monitor all accounts)
* Channels (unique to the Python SDK)


### Accessing the Kin Blockchain

The SDK has two main components, `KinClient` and `KinAccount`.

- **KinClient** is used to query the blockchain and perform actions that don't require authentication (e.g get an account balance)  
- **KinAccount** is used to perform authenticated actions on the blockchain (e.g send payment)

To initialize `KinClient` you will need to provide an environment (Test and Production environments are preconfigured).


```python
from kin import KinClient, TEST_ENVIRONMENT

client = KinClient(TEST_ENVIRONMENT)
```

Or you can configure a custom environment with your own parameters:  
```python
from kin import Environment

MY_CUSTOM_ENVIRONMENT = Environemnt('name','horizon endpoint','network passphrase','friendbot url'(optional))
```

Once you have a KinClient, you can use it to get a KinAccount object.

The KinAccount object can be initialized in two ways:

```python
# With a single seed:
account = client.kin_account('seed')

# With channels:
account = client.kin_account('seed', channel_secret_keys=['seed1','seed2','seed3'...])

# Additionally, a unique app-id can be provided, this will be added to all your transactions
account = client.kin_account('seed',app_id='unique_app_id')
```
Read more about channels in the [Channels](#Channels) section.

`appID` is currently assigned as part of the Kin Developer Program. This will be automated in the future.

### Checking configuration
The handy `get_config` method will return some parameters the client was configured with, along with Horizon status:
```python
# Disclaimer: the below JSON is outdated
status = client.get_config()
print status
{
  "horizon": {
    "uri": "https://horizon-playground.kininfrastructure.com",
    "online": true,
    "error": null
  },
  "sdk_version": "2.0.0",
  "environment": "PLAYGROUND",
  "kin_asset": {
    "code": "KIN",
    "issuer": "GBC3SG6NGTSZ2OMH3FFGB7UVRQWILW367U4GSOOF4TFSZONV42UJXUH7"
  },
  "transport": {
    "pool_size": 10,
    "request_timeout": 11,
    "backoff_factor": 0.5,
    "num_retries": 5,
    "retry_statuses": [
      503,
      413,
      429,
      504
    ]
  }
}
```
- `sdk_version` - the version of this SDK.
- `address` - the SDK wallet address.
- `kin_asset` - the KIN asset the SDK was configured with.
- `environment` - the environment the SDK was configured with (TEST/PROD/CUSTOM).
- `horizon`:
  - `uri` - the endpoint URI of the Horizon server.
  - `online` - Horizon online status.
  - `error` - Horizon error (when not `online`) .
- `transport`:
  - `pool_size` - number of pooled connections to Horizon.
  - `num_retries` - number of retries on failed request.
  - `request_timeout` - single request timeout.
  - `retry_statuses` - a list of statuses to retry on.
  - `backoff_factor` - a backoff factor to apply between retry attempts.


### Managing Kin accounts
Most methods provided by `KinClient` to query the blockchain about a specific account can also be used from the KinAccount object to query the blockchain about itself.

#### Creating and retrieving a Kin account

The very first thing we need to do before you can send or receive Kin is creating an account on the blockchain. This is how you do it:

```python
# the KIN amount can be specified in numbers or as a string
tx_hash = account.create_account('address', starting_balance=1000, fee=100)

# a text memo can also be added; memos cannot exceed 21 characters:
tx_hash = account.create_account('address', starting_balance=1000, fee=100, memo_text='My first account')
```

#### Account Details
 Each account on the Kin Blockchain has an associated public address and a secret seed (often also referred as public and private key).
```python
address = account.get_public_address()
```

#### Checking if an account exists on the blockchain
There is one thing you can do even without an account object: check to see  if an thataccount already exists on the blockchain.

```python
client.does_account_exists('address')
```

#### Account balance and data
Now that you have an account you can check its balance.

```python
balance = client.get_account_balance('address')
```

There is of course a lot more about an account besides its balance. You can get more information with `get_account_data`.

```python
account_data = client.get_account_data('address')
```

The output will look something like this:

```
{'_data':
  {
    'id': 'GDNGBE7S3ZHXAUAGSMVDNUM2FRTIRNDT3QHRMR5CVPI4YYSLL5ZUM2ME',
    'account_id': 'GDNGBE7S3ZHXAUAGSMVDNUM2FRTIRNDT3QHRMR5CVPI4YYSLL5ZUM2ME',
    'sequence': '15167341998374912',
    'data': {},
    'thresholds': 	_data='{
      'low_threshold': 0,
      'med_threshold': 0,
      'high_threshold': 0
    }',
    'balances': [	_data='{
      asset_type': 'native',
      'asset_code': None,
      'asset_issuer': None,
      'balance': 100.0,
      'limit': None
    }'],
    'flags': 	_data='{
      'auth_required': False,
      'auth_revocable': False
    }',
    'paging_token': '',
    'subentry_count': 0,
    'signers': [	_data='{
      'public_key': 'GDNGBE7S3ZHXAUAGSMVDNUM2FRTIRNDT3QHRMR5CVPI4YYSLL5ZUM2ME',
      'key': 'GDNGBE7S3ZHXAUAGSMVDNUM2FRTIRNDT3QHRMR5CVPI4YYSLL5ZUM2ME',
      'weight': 1,
      'signature_type':
      'ed25519_public_key'
    }']
  }
}
```

### Get account status
Often times you will want to know the status of an account; you can do this easily with `get_status`. The function expects a single parameter, boolean, if set to `True` all channels and statuses will be printed.

```python
account.get_status(True)
```

```json
{
  "client": {
    "sdk_version": "2.2.0",
    "environment": "LOCAL",
    "horizon": {
      "uri": "http://localhost:8000",
      "online": true,
      "error": null
    },
    "transport": {
      "pool_size": 10,
      "num_retries": 5,
      "request_timeout": 11,
      "retry_statuses": [
        503,
        413,
        429,
        504
      ],
      "backoff_factor": 0.5
    }
  },
  "account": {
    "app_id": "anon",
    "public_address": "GCLBBAIDP34M4JACPQJUYNSPZCQK7IRHV7ETKV6U53JPYYUIIVDVJJFQ",
    "balance": 9999989999199.979,
    "channels": {
      "total_channels": 5,
      "free_channels": 4,
      "non_free_channels": 1,
      "channels": {
        "SBS3O5BGCPDIYWTTOV7TGLXFRPFSD6ACBEAEHJUMMPF5DUDF732MX6LL": "free",
        "SC65CIJCAWJEJX5IVHDJK6FO6DM5BVPIUX5F7EULIC3C4PF7KTAUHHE2": "free",
        "SABWFQ2HOYPQGCWN7INIV2RNZZLAZDOX67R3VHMGQAFF6FA3JIA2E7BB": "free",
        "SBBQJTYF6K2TDUJ2LBUSXICUEEX75RXAQZRP6LLVF3JDXK5D4SVYX3X4": "taken",
        "SCD36QIV3SFEGZDHRZZXO7MICNMOHSRAOV6L2MQKSW4TO4OTCR4IF2FD": "free"
      }
    }
  }
}
```

#### Keypairs
Earlier we talked about public address and secret seed. Here are a few convenient functions to generate the keypairs.

###### Create a new keypair
```python
from kin import Keypair

my_keypair = Keypair()
# Or, you can create a keypair from an existing seed
my_keypair = Keypair('seed')
```

###### Getting the public address from a seed
```python
public_address = Keypair.address_from_seed('seed')
```

###### Generate a new random seed
```python
seed = Keypair.generate_seed()
```

###### Generate a deterministic seed
```python
# Given the same seed and salt, the same seed will always be generated
seed = Keypair.generate_hd_seed('seed','salt')
```

###### Generate a mnemonic seed:
**Not yet implemented**

### Transactions
Transactions are executed on the Kin Blockchain in a two-step process.

* **Build** the transaction, including calculation of the transaction hash. The hash is used as a transaction ID and is necessary to query the status of the transaction.
* **Send** the transaction to servers for execution on the blockchain.


#### Transferring Kin to another account
To transfer Kin to another account, you need the public address of the account to which you want to transfer Kin.

By default, your user will need to spend Fee to transfer Kin or process any other blockchain transaction. Fee for individual transactions are trivial 1 Kin = 10E5 Fee.

Some apps can be added to the Kin whitelist, a set of pre-approved apps whose users will not be charged Fee to execute transactions. If your app is in the whitelist then refer to [Transferring Kin to another account using whitelist service](#transferring-kin-to-another-account-using-whitelist-service).

Below is an example of how to transfer 20 Kin to a destination address and paying 100 Fee in just one line:
```python
destination = 'GDIRGGTBE3H4CUIHNIFZGUECGFQ5MBGIZTPWGUHPIEVOOHFHSCAGMEHO'
# the KIN amount can be specified in numbers or as a string
tx_hash = account.send_kin(destination, 20, fee=100, memo_text='Thank you Kin')
```

The above example will work in most cases, but if you need to split the process in multiple steps you can first build the transaction, update any parameters and then execute.

Step 1: Build the transaction
```python
builder = account.build_send_kin(destination, 20, fee=100, memo_text='tx in 3-steps')
```

Step 2: Update the transaction
```python
# Configure additional parameters
with account.channel_manager.get_channel() as channel:
    builder.set_channel(channel)
    builder.sign(channel)
    # If you used additional channels apart from your main account,
    # sign with your main account
    builder.sign(account.keypair.secret_seed)
```
Step 3: Send the transaction
```python
    tx_hash = account.submit_transaction(builder)
```

#### Transferring Kin to another account using whitelist service
The Kin Blockchain also allows for transactions to be executed with no fee. If your service has been added to the this list you will be able to whitelist transactions for your clients.

Clients will send an http request to your Python app containing their transaction, you can then whitelist it and return it to the client to send to the blockchain.

```python
whitelisted_tx = account.whitelist_transaction(client_transaction)
```

Please note that if you are whitelisted, any payment sent from you (an app developed with the Python SDK) is already considered whitelisted, so there is no need for this step for the server transactions.

### Decode_transaction
When clients send you transactions for whitelisting they will be encoded. You can use `decode_transaction` to read and then verify the contents.

```python
from kin import decode_transaction

decoded_tx = decode_transaction(encoded_tx)
```

#### Getting the minimum acceptable fee from the blockchain
Transactions usually require a fee to be processed. In most examples we used a fixed fee, but a smarter way is to query the blockchain for the minimum fee.

```python
minimum_fee = client.get_minimum_fee()
```

Trying to send a transaction with insufficient Fee will result in a failed transaction. The blockchain will return an error for the developer to manage.

#### Getting Transaction Data
Often times you will want to review a transaction, `get_transaction_data` is here to help you.

The function is quite simple and expects a transaction hash and a second parameter called `simple`:
* **True** will return the object `kin.RawTransaction`, this is good for debugging and testing, but not for user messages
* **False** will return the object `kin.SimpleTransaction`, this is likely OK to be formatted and showed to end users. Notice: if the transaction if too complex to be simplified, a `CantSimplifyError` will be raised

```python
tx_data = sdk.get_transaction_data(tx_hash, True)
```

### Friendbot
If a friendbot endpoint is provided when creating the environment (it is provided with the TEST_ENVIRONMENT), you will be able to use the friendbot method to call a service that will create an account for you. Friendbot is useful for example to fund accounts in the test environment.

```python
client.friendbot('address')
```


## Monitoring Kin Payments
These methods can be used to monitor Kin payments that an account or multiple accounts are sending or receiving.

Kin SDKs designed for clients such as iOS and Android can monitor the accounts associated with their local users. The Python SDK can monitor other users' accounts. This is currently unique to the Python SDK.

**Currently, due to a bug on the blockchain frontend, the monitor may also return 1 transaction that happened before the monitoring request**


The monitor will run in a background thread (accessible via `monitor.thread`), and will call the callback function every time it finds a Kin payment for the given address.

### Monitor a single account
Monitoring a single account will continuously get data about this account from the blockchain and filter it.

```python
def callback_fn(address, tx_data, monitor)
	print ('Found tx: {} for address: {}'.format(address,tx_data.id))

monitor = client.monitor_account_payments('address', callback_fn)
```

### Monitor multiple accounts
It is possible to monitor multiple accounts using `monitor_accounts_payments`, the function will continuously get data about **all** accounts on the blockchain, and will filter for the selected accounts.

```python
def callback_fn(address, tx_data, monitor)
	print ('Found tx: {} for address: {}'.format(address,tx_data.id))

monitor = client.monitor_accounts_payments(['address1','address2'], callback_fn)
```

You can freely add or remove accounts to this monitor

```python
monitor.add_address('address3')
monitor.remove_address('address1')
```

### Stopping a monitor
When you are done monitoring, make sure to stop the monitor, to terminate the thread and the connection to the blockchain.

```python
monitor.stop()
```


## Channels

The Kin Blockchain is based on the the Stellar blockchain. One of the most sensitive points in Stellar is [transaction sequence](https://www.stellar.org/developers/guides/concepts/transactions.html#sequence-number).
In order for a transaction to be submitted successfully, this number should be correct. However, if you have several SDK instances, each working with the same wallet account or channel accounts, sequence collisions will occur.

We highly recommend to keep only one KinAccount instance in your application, having unique channel accounts.
Depending on the nature of your application, here are our recommendations:

Here are our recommendations based on how you configure your server(s):

### Simple script

If you have a simple (command line) script that sends transactions on demand or only once in a while, we recommend you create one instance of `KinAccount` with only the SDK wallet key. Channel accounts are not necessary.

### Single application server

If you have a single application server that handles a stream of concurrent transactions, we recommend you create one instance of `KinAccount` initialized with multiple channel accounts.

**Note:** If you use a standard `gunicorn/Flask` setup, Gunicorn will spawn several *worker processes*. Each process will contain your Flask application and therefore your `KinAccount` instance, so multiple `KinAccount` instances will exist with identical channel accounts.

The solution is to use Gunicorn *thread workers* instead of *process workers*. For example if you run Gunicorn with the `--threads` switch instead of `--workers`, only one Flask application is created containing a single `KinAccount` instance.

### Load-balanced application servers

If you have a number of load-balanced application servers each server should:

- be set up the same as a [single application server](#single-application-server)
- have its own channel accounts

This ensures you will not have any collisions in your transaction
sequences.

### Creating Channels
The Kin SDK for Python allows you to create HD (highly deterministic) channels based on your seed and a passphrase to be used as a salt. As long as you use the same seed and passphrase, you will always get the same seeds.

```python
import kin.utils

channels = utils.create_channels(master_seed, environment, amount, starting_balance, salt)
```

`channels` will be a list of seeds the sdk created for you, that can be used when initializing the KinAccount object.

If you just wish to get the list of the channels generated from your seed + passphrase combination without creating them

```python
channels = utils.get_hd_channels(master_seed, salt, amount)
```


## License
The code is currently released under [MIT license](LICENSE).