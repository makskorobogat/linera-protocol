# Matching Engine Example Application

This sample application demonstrates a matching engine, showcasing the DeFi capabilities
on the Linera protocol.

The matching engine trades between two tokens `token_0` & `token_1`. We can refer to
the `fungible` application example on how to create two token applications.

An order can be of two types:

- Bid: For buying token 1 and paying in token 0, these are ordered in from the highest
  bid (most preferable) to the lowest price.
- Ask: For selling token 1, to be paid in token 0, these are ordered from the lowest
  (most preferable) to the highest price.

An `OrderId` is used to uniquely identify an order and enables the following functionality:

- Modify: Allows to modify the order.
- Cancel: Cancelling an order.

When inserting an order it goes through the following steps:

- Transfer of tokens from the `fungible` application to the `matching engine` application through a cross-application
  call so that it can be paid to the counterparty.

- The engine selects the matching price levels for the inserted order. It then proceeds
  to clear these levels, executing trades and ensuring that at the end of the process,
  the best bid has a higher price than the best ask. This involves adjusting the orders in the market
  and potentially creating a series of transfer orders for the required tokens. If, after
  the level clearing, the order is completely filled, it is not inserted. Otherwise,
  it becomes a liquidity order in the matching engine, providing depth to the market
  and potentially being matched with future orders.

When an order is created from a remote chain, it transfers the tokens of the same owner
from the remote chain to the chain of the matching engine, and a `ExecuteOrder` message is sent with the order details.

## Usage

### Setting Up

Before getting started, make sure that the binary tools `linera*` corresponding to
your version of `linera-sdk` are in your PATH. For scripting purposes, we also assume
that the BASH function `linera_spawn_and_read_wallet_variables` is defined.

From the root of Linera repository, this can be achieved as follows:

```bash
export PATH="$PWD/target/debug:$PATH"
source /dev/stdin <<<"$(linera net helper 2>/dev/null)"
```

To start the local Linera network:

```bash
linera_spawn_and_read_wallet_variables linera net up --testing-prng-seed 37
```

We use the test-only CLI option `--testing-prng-seed` to make keys deterministic and simplify our
explanation.

```bash
OWNER_1=de166237331a2966d8cf6778e81a8c007b4084be80dc1e0409d51f216c1deaa1
OWNER_2=38e7f6b6d1887b0ea1511e04a08382577aaf551bb440ee6011966d73d19b8150
OWNER_3=789dcf5afb0afb9c5c473568bab33c68b83005bf7b964dc2a93e5d4368a9da1f
CHAIN_1=aee928d4bf3880353b4a3cd9b6f88e6cc6e5ed050860abae439e7782e9b2dfe8
CHAIN_2=a3edc33d8e951a1139333be8a4b56646b5598a8f51216e86592d881808972b07
PUB_KEY_1=fcf518d56455283ace2bbc11c71e684eb58af81bc98b96a18129e825ce24ea84
PUB_KEY_2=ca909dcf60df014c166be17eb4a9f6e2f9383314a57510206a54cd841ade455e
```

Publish and create two `fungible` applications whose IDs will be used as a
parameter for the `matching-engine` application. The flag `--wait-for-outgoing-messages`
waits until a quorum of validators has confirmed that all sent cross-chain messages have been
delivered.

```bash
FUN1_APP_ID=$(linera --wait-for-outgoing-messages \
  project publish-and-create examples/fungible \
    --json-argument "{ \"accounts\": {
        \"User:$OWNER_1\": \"100.\",
        \"User:$OWNER_2\": \"150.\"
    } }" \
    --json-parameters "{ \"ticker_symbol\": \"FUN1\" }" \
)

FUN2_APP_ID=$(linera --wait-for-outgoing-messages \
  project publish-and-create examples/fungible \
    --json-argument "{ \"accounts\": {
        \"User:$OWNER_1\": \"100.\",
        \"User:$OWNER_2\": \"150.\"
    } }" \
    --json-parameters "{ \"ticker_symbol\": \"FUN2\" }" \
)
```

Now we publish and deploy the Matching Engine application:

```bash
MATCHING_ENGINE=$(linera --wait-for-outgoing-messages \
    project publish-and-create examples/matching-engine \
    --json-parameters "{\"tokens\":["\"$FUN1_APP_ID\"","\"$FUN2_APP_ID\""]}" \
    --required-application-ids $FUN1_APP_ID $FUN2_APP_ID)
```

And make sure chain 2 also has it:

```bash
linera --wait-for-outgoing-messages request-application \
    --requester-chain-id $CHAIN_2 $MATCHING_ENGINE
```

### Using the Matching Engine Application

First, a node service for the current wallet has to be started:

```bash
PORT=8080
linera service --port $PORT &
```

#### Using GraphiQL

Type each of these in the GraphiQL interface and substitute the env variables with their actual values that we've defined above.

Navigate to the URL you get by running `echo "http://localhost:8080/chains/$CHAIN_1/applications/$MATCHING_ENGINE"`:

To create a `Bid` order as owner 1, offering to buy 1 FUN1 for 5 FUN2:

```gql,uri=http://localhost:8080/chains/$CHAIN_1/applications/$MATCHING_ENGINE
mutation {
  executeOrder(
    order: {
        Insert : {
        owner: "User:$OWNER_1",
        amount: "1",
        nature: Bid,
        price: {
            price:5
        }
      }
    }
  )
}
```

To query about the bid price:

```gql,uri=http://localhost:8080/chains/$CHAIN_1/applications/$MATCHING_ENGINE
query {
  bids {
    keys {
      price
    }
  }
}
```

#### Atomic Swaps

In general, if you send tokens to a chain owned by someone else, you rely on them
for asset availability: If they don't handle your messages, you don't have access to
your tokens.

Fortunately, Linera provides a solution based on temporary chains:
If the number of parties who want to swap tokens is limited, we can make them all chain
owners, allow only Matching Engine operations on the chain, and allow only the Matching
Engine to close the chain.

```bash
kill %% && sleep 1    # Kill the service so we can use CLI commands for chain 1.

linera --wait-for-outgoing-messages change-ownership \
    --owners $OWNER_1 $OWNER_2

linera --wait-for-outgoing-messages change-application-permissions \
    --execute-operations $MATCHING_ENGINE \
    --close-chain $MATCHING_ENGINE

linera service --port $PORT &
```

First, owner 2 should claim their tokens. Navigate to the URL you get by running `echo "http://localhost:8080/chains/$CHAIN_2/applications/$FUN1_APP_ID"`:

```gql,uri=http://localhost:8080/chains/$CHAIN_2/applications/$FUN1_APP_ID
mutation {
  claim(
    sourceAccount: {
      chainId: "$CHAIN_1",
      owner: "User:$OWNER_2",
    }
    amount: "100.",
    targetAccount: {
      chainId: "$CHAIN_2",
      owner: "User:$OWNER_3"
    }
  )
}
```

And to the URL you get by running `echo "http://localhost:8080/chains/$CHAIN_2/applications/$FUN2_APP_ID"`:

```gql,uri=http://localhost:8080/chains/$CHAIN_2/applications/$FUN2_APP_ID
mutation {
  claim(
    sourceAccount: {
      chainId: "$CHAIN_1",
      owner: "User:$OWNER_2",
    }
    amount: "150.",
    targetAccount: {
      chainId: "$CHAIN_2",
      owner: "User:$OWNER_2"
    }
  )
}
```

Owner 2 offers to buy 2 FUN1 for 10 FUN2. This gets partially filled, and they buy 1 FUN1
for 5 FUN2 from owner 1. This leaves 5 FUN2 of owner 2 on chain 1. On the URL you get by running `echo "http://localhost:8080/chains/$CHAIN_2/applications/$MATCHING_ENGINE"`:

```gql,uri=http://localhost:8080/chains/$CHAIN_2/applications/$MATCHING_ENGINE
mutation {
  executeOrder(
    order: {
      Insert : {
        owner: "User:$OWNER_2",
        amount: "2",
        nature: Ask,
        price: {
            price:5
        }
      }
    }
  )
}
```

The only way to close the chain is via the application now. On the URL you get by running `echo "http://localhost:8080/chains/$CHAIN_1/applications/$MATCHING_ENGINE"`:

```gql,uri=http://localhost:8080/chains/$CHAIN_1/applications/$MATCHING_ENGINE
mutation { closeChain }
```

Owner 2 should now get back their tokens, and have 145 FUN2 left. On the URL you get by running `echo "http://localhost:8080/chains/$CHAIN_2/applications/$FUN2_APP_ID"`

```gql,uri=http://localhost:8080/chains/$CHAIN_2/applications/$FUN2_APP_ID
query {
  accounts {
    entry(
      key: "User:$OWNER_2"
    ) {
      value
    }
  }
}
```
