# Aries - Endorser Service

This repository provides an Endoser agent, based on [Aries Cloudagent Python (or Aca-Py)](https://github.com/hyperledger/aries-cloudagent-python).

Information about Aca-Py's Endorser support can be found [here](https://github.com/hyperledger/aries-cloudagent-python/blob/main/Endorser.md).

The Aca-Py Alice/Faber demo also demonstrates the use of the Endorser feature, as described [here](https://github.com/hyperledger/aries-cloudagent-python/blob/main/demo/Endorser.md).

This repository is a work in progress, see [this document](https://hackmd.io/hWMLdpu7SBuopNag4mTbcg?view) for the on-going requirements and design.

## Running Locally

Prerequisites - you can run a local von-network and tails server, or connect to an existing service.

To run everything locally, open 2 bash shells and run the following:

```bash
git clone https://github.com/bcgov/von-network.git
cd von-network
./manage build
./manage run --logs
```

```bash
git clone https://github.com/bcgov/indy-tails-server.git
cd indy-tails-server/docker
./manage build
./manage run --logs
```

Then, to get the endorser service up and running quicky, open a bash shell and run the following:

```bash
git clone https://github.com/bcgov/aries-endorser-service.git
cd aries-endorser-service/docker
./manage build
./manage start --logs
```

You can open the Endorser Admin API in your browser at http://localhost:5050/endorser/docs - you will need to authenticate using the configured id and password (endorser-admin/change-me).  (The webhooks api is at http://localhost:5050/webhook/docs, although you shouldn't need to use this one directly.)

To shut down the service:

```bash
<ctrl-c>
./manage rm
```

By default, the Endorser runs with a local ledger and tails server.

To run against the BCovrin Test ledger (http://test.bcovrin.vonx.io/) and tails server (https://tails-test.vonx.io), start the endorser using the `LEDGER_URL` and `TAILS_SERVER_URL` parameters:

```bash
LEDGER_URL=http://test.bcovrin.vonx.io TAILS_SERVER_URL=https://tails-test.vonx.io ./manage start --logs
```

You can also use the `GENESIS_URL` parameter to run against a non-von-network ledger.  For example, the SOVRIN staging ledger's transactions can be found [here](https://raw.githubusercontent.com/sovrin-foundation/sovrin/master/sovrin/pool_transactions_sandbox_genesis).

```bash
GENESIS_URL=https://raw.githubusercontent.com/sovrin-foundation/sovrin/master/sovrin/pool_transactions_sandbox_genesis ./manage start --logs
```

(And obviously with the above you need to provide a suitable tails server parameter.)

By default, the `./manage` script will use a random seed to generate the Endorser's public DID.  Author agents will need to know this public DID in order to create transactions for endorsement.  If you need to start the Endorser using a "well known DID" you can start with the `ENDORSER_SEED` parameter:

```bash
ENDORSER_SEED=<your 32 char seed> ./manage start --logs
```

## Endorser Configuration

There are 3 "global" configuration options that can be set using environment variables, or can be set using the Endorser Admin API, the environment variables are:

- `ENDORSER_AUTO_ACCEPT_CONNECTIONS`: set to `true` for the Endorser service to auto-accept connections (otherwise they must be manually accepted)
- `ENDORSER_AUTO_ACCEPT_AUTHORS`: set to `true` for the Endorser service to auto-configure new connections to be "authors", otherwise author meta-data must be manually set
- `ENDORSER_AUTO_ENDORSE_REQUESTS`: set to `true` for the Endorser service to auto-accept all "endorse transaction" requests (otherwise they must be manually endorsed) 

These parameters can be set using the `POST /endorser/v1/admin/config/<env var name>?config_value=<value>` admin API, and the setting is stored in the database (the database setting will override the environment variable).  You can see the configured values using the `GET /endorser/v1/admin/config/<env var name>` endpoint (and it will let you know if the configuration is using a value from the database or environment variable).

There are 2 endpoints to set connection-specific (i.e. author-specific) configuration.

`PUT /endorser/v1/connections/<connection_id>` - sets the alias and (optionally) public DID for the author (not currently used anywhere by the Endorser, but may be useful).

`PUT /endorser/v1/connections/<connection_id>/configure` - sets processing options for the connection:

- `author_status` - `Active` or `Suspended` - if not Active, all requests from this connection will be ignored
- `endorse_status` - `AutoEndorse`, `ManualEndorse` or `AutoReject` - the "auto" options will automatically endorse or refuse endorsement requests (respectively), for the "manual" option the requests must be manually endorsed

Endorsement requests will be auto-endorsed if the `ENDORSER_AUTO_ENDORSE_REQUESTS` setting is `true` *or* if the `endorse_status` is set to `AutoEndorse` on the connection.  So, if manual endorsements are desired, `ENDORSER_AUTO_ENDORSE_REQUESTS` should be set to `false` *and* each connection should be set to `ManualEndorse` (which is the default).


## Testing - Integration tests using Behave

This repository includes integration tests implemented using Behave.

When you start the endorser service, you can optionally start an additional Author agent which is used for testing:

```bash
./manage start-bdd --logs
```

The Author agent (which is configured as multi-tenant) exposes its Admin API on http://localhost:8061/api/doc

Open a second bash shell (cd to the directory where you have checked out this repository) and run:

```bash
virtualenv venv
source ./venv/bin/activate
pip install -r endorser/requirements.txt
cd docker
```

(Note that the above are one-time commands)

To run the BDD tests just run:

```bash
LEDGER_URL=http://localhost:9000 TAILS_SERVER_URL=http://localhost:6543 ./manage run-bdd
```

... or to run a specific test (or group of tests), for example:

```bash
LEDGER_URL=http://localhost:9000 TAILS_SERVER_URL=http://localhost:6543 ./manage run-bdd -t @DIDs-006
```

Note that because these tests run on your local (rather than in a docker container) you need to specify the *local* url to the ledger and tails servers.


## Testing - Other

You can also test using [traction](https://github.com/bcgov/traction).

(Note that this describes the "old" traction integration tests.  There are new tests in traction using behave, this doc should be updated to reflect the latest traction.)

Open a bash shell and startup the endorser services:

```bash
git clone https://github.com/bcgov/aries-endorser-service.git
cd aries-endorser-service/docker
./manage build
# note we start with a "known" endorser DID
# ... and we need to point to the same ledger and tails server as traction ...
ENDORSER_SEED=testendorserseed_123123123123123 LEDGER_URL=<...> TAILS_SERVER_URL=<...> ./manage start --logs
```

Then open a separate bash shell and run the following:

```bash
git clone https://github.com/bcgov/traction.git
cd traction/scripts
git checkout endorser-integration
cp .env-example .env
docker-compose build
docker-compose up
```

Then open up yet another bash shell and run the traction integration tests.  Traction tenants will connect to the endorser service for creating schemas, credential definitions etc.

```bash
cd <traction/scripts directory from above>
docker exec scripts_traction-api_1 pytest --asyncio-mode=strict -m integtest
```

... or you can run individual tests like this:

```bash
docker exec scripts_traction-api_1 pytest --asyncio-mode=strict -m integtest tests/integration/endpoints/routes/test_tenant.py::test_tenant_issuer
```

Traction integration tests create new tenants for each test, so you can rebuild/restart the endorser service without having to restart traction.
