# Get price data

## Create a broker account and api keys

Create a broker account for a broker that provides a price data API.

Your broker account should provide some API keys that are required to connect. Save these somewhere safe for now.

## Create a vault and store your API keys

On your IPA server, install the KRA service. Your pki service will need to be running

```
sudo ipactl start
sudo ipa-kra-install
```

Create the vault. The following will create a vault for connecting to an alpaca paper trading account.

```
ipa vault-add alpaca-paper --type=standard
```

Save the api connection info in your vault. First save them into a file, then load the file into the vault and delete the file

```
echo "url=https://paper-api.alpaca.markets" > alpaca-paper
echo "key=[my api key]" >> alpaca-paper
echo "secret_key=[my api secret key]" >> alpaca-paper
cat alpaca-paper
ipa vault-archive alpaca-paper --in=alpaca-paper
rm alpaca-paper
```

## Get the vault from your lab

On your lab, launch Terminal

Retrieve the vault data into a file named .vault in your notebooks directory.

```
ipa vault-retrieve alpaca-paper  --out ~/lab/notebooks/.vault
```

You can then use the data in the vault to connect to your broker and subscribe to price data updates. Example Python script below.

```
from alpaca_trade_api.stream import Stream
from alpaca_trade_api.common import URL
import nest_asyncio

# Load vault file as dict
con_params = {}
with open(".vault") as f:
    for line in f:
       (key, val) = line.split("=")
       con_params[key] = val.rstrip()
       
# Only needed in Jupyter
nest_asyncio.apply()

async def trade_callback(t):
    print('trade', t)
async def quote_callback(q):
    print('quote', q)
# Initiate Class Instance
stream = Stream(con_params['key'],
                con_params['secret-key'],
                base_url=con_params['url'],
                data_feed='iex')  # <- replace to SIP if you have PRO subscription
# subscribing to event
stream.subscribe_trades(trade_callback, 'AAPL')
stream.subscribe_quotes(quote_callback, 'IBM')
stream.run()
```
