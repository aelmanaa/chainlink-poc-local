# Chainlink local


## Local node

This proof of concept explains how to run a chainlink node on your computer. This should not be used as production environment but rather as an exercice to get familiar with chainlink concepts.

### Prerequisites

* Must have docker installed [docker](https://docs.docker.com/get-docker/)
* Must have a connection to Ethereum , either using a local client or external provider. In my case, I'm using [Infura](https://infura.io/)

### Run a Chainlink Node

#### Create a docker network bridge

We are going to run 2 containers:
* One for Chainlink node
* One for Postgresql

Chainlink node must be able to access postgresql container. By defualt, every container is isolated. Unless we define a  network bridge and configure containers to use this network.

```
 docker network create cl-net
```

#### Run a local Postgresql database

We are running here the [official postgres docker image](https://hub.docker.com/_/postgres). Note the user name is `cl-node` and password is `cl-node-pass` . we are also mounting a volume in order to persist data even if the container restarts

```
mkdir ./pg-data

docker run --name cl-postgres -e POSTGRES_DB=postgres -e POSTGRES_USER=cl-node --network cl-net -e POSTGRES_PASSWORD=cl-node-pass -e PGDATA=/var/lib/postgresql/data/pgdata \
-p 5432:5432 \
-v pg-data:/var/lib/postgresql/data \
-d postgres:13.3
```

Note that network is passed to the commande line `--network cl-net`

At this point, a postgres instance is running on your local machine on port `5432`

you can check that the postgres image is running

```
docker ps
```

Command should display `cl-postgres` as "UP"

![PG docker successful](images/docker-ps-pg-success.png)

#### Install a postgresql client

This is optional but you can install a postgresql client in order to view data that will be created. I'm using [pgadmin](https://www.postgresql.org/)

Once installed, you can create a new connection:
* username : cl-node
* database: postgres
* password: cl-node-pass
* server url: localhost
* port: 5432

Once saved, connection should be sucessful and you'll be able to see your database as show below:
![DB succesful connection](images/pgadmin-success.png)

#### Prepare environment files

Follow the steps [Running a Chainlink Node](https://docs.chain.link/docs/running-a-chainlink-node/)

Remark: don't forget to mention "sslmode=disable" in the postgres connection string . For instance, if you chose to test on Rinkeby then your config should look like:

```
echo "DATABASE_URL=postgresql://cl-node:cl-node-pass@cl-postgres:5432/postgres?sslmode=disable" >> ~/.chainlink-rinkeby/.env
```

Note `cl-postgres` is the container name of postgres. As we will run chainlink container on the same network, "cl-postgres" will be resolved to an internal IP reachable by chainlink container

#### Run Chainlink node

When running the chainlink node, do not forget to specify the network so that chainlink node is able to call Postgres container. Your command should look like the following:

```
cd ~/.chainlink-rinkeby && docker run --name chainlink -p 6688:6688 --network cl-net -v ~/.chainlink-rinkeby:/chainlink -it --env-file=.env smartcontract/chainlink:0.10.8 local n -p /chainlink/.password -a /chainlink/.api
```

Now running `docker ps` should display 2 containers

![Chainlink docker successful](images/chainkink-success.png)

#### Import account in Metamask

sometimes it could be interesting to view the account in metamask and be able to check the funds. To do so, please follow these steps:

* exec inside your container `docker exec -it chainlink bash`
* go inside the host mounted folder `cd /chainlink`
* login using username and password that was set in ".api" files (cfr. [Running a Chainlink Node](https://docs.chain.link/docs/running-a-chainlink-node/)) : `chainlink admin login`
* you can view imported accounts using: `chainlink keys eth list`
* you can export the node account key using the password that was set in ".password" file (cfr. [Running a Chainlink Node](https://docs.chain.link/docs/running-a-chainlink-node/))  `chainlink keys eth export 0xCD5f012fdFFf3d653085e5A41a7F9356A0D0b733 -p .password --output key.json`
* on the host machine, you should find "key.json" in "~/.chainlink-rinkeby" (If you are using rinkeby for your tests)
* you can import "key.json" in metamask. If will ask you for a password, you can use the password that was set in ".password"

#### Fund the node account

you can get ETH and LINK following these links:

* [LINK](https://docs.chain.link/docs/acquire-link/)
* list of ETH faucets  per environment could be found here: [ETH](https://support.mycrypto.com/how-to/getting-started/where-to-get-testnet-ether)

#### Test the node 

you can start fulfilling requests following this [chianlink tutorial](https://docs.chain.link/docs/fulfilling-requests/)

## Chainlink external adapter

based on chainlink blog: [Building and using External Adapters](https://blog.chain.link/build-and-use-external-adapters/)

### How to build an externa adapter

1st part of blog

```
cd external-adapter/eth-price
yarn
yarn start
```

in another terminal , you should get a succesful reponse if you run 
`curl -X POST -H "content-type:application/json" "http://localhost:8080/" --data '{ "id": 0, "data": { "from": "ETH", "to": "USD" } }'`

### Getting Weather Data

2nd part of blog
* You need first to register in [Openweathermap](https://openweathermap.org/) and get an API key. 
* Create a `.env`file then add the API Key to it `API_KEY=<YOUR_API_KEY>`

```
cd external-adapter/weather
yarn
yart start
```

in another terminal , you should get a succesful reponse if you run 
`curl -X POST -H "content-type:application/json" "http://localhost:8080/" --data '{ "id": 0, "data": { "city": "Boston"} }`
