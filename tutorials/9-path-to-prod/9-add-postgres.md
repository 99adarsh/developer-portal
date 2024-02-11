## Run the node with PostgreSQL Database for indexed data

In this tutorial, we will learn how to run a cosmos chain node(we will run cosmoshub node in this example) in Ubuntu OS with postgres database configuration to store indexed data in postgres database, and query the table to get all the transactions from a block.


CometBft provides the capability to index blocks and transactions for subsequent querying. One can query blocks and transaction data using a node's JSON-RPC or by running their node and querying the data from the database. We can see all the available rpc specifications [here](https://github.com/cometbft/cometbft/blob/main/spec/rpc/README.md).

There are three options to configure cometbft for indexing - null, kv, and psql.

1. null - Node will not index the data
2. kv - kv is used by default for indexing, the node stores data in key-value pairs(using levelDB)
3. psql - Node can store the blocks and transaction data in Postgres table.

Checkout [the hardware requirements](https://hub.cosmos.network/main/hub-tutorials/join-mainnet.html#hardware) for running a node in your system, we will be running a pruned node (pruned node only stores data for some past blocks, it is configurable).

**Steps**

1. **Install Gaia binary**

   - Follow [this guide](https://hub.cosmos.network/main/getting-started/installation.html) to install gaia binary in your system.

2. **Initialize chain**

   - Initialize the chain using the below command and provide a human readable name as custom-moniker. It will create `~/.gaia` directory with `config` and `data` subfolders.

       ```golang
       gaiad init <custom-moniker>
       ```


3. **Download genesis**

   - Download the genesis file for cosmoshub and move it to the `~/.gaia/config` directory. You can find the genesis file for other cosmos chains on [polkachu](https://polkachu.com/genesis).
   - The below command will download the cosmoshub's genesis file, unzip it, and move it to `~/.gaia/config` directory.

       ```
       wget https://raw.githubusercontent.com/cosmos/mainnet/master/genesis/genesis.cosmoshub-4.json.gz
       gzip -d genesis.cosmoshub-4.json.gz
       mv genesis.cosmoshub-4.json ~/.gaia/config/genesis.json
       ```

4. **Configure config.toml for adding seed and peers**

   - After initialization, the node will have to connect to the peers to be a part of the chain. We can find active peer information on [polkachu](https://polkachu.com/live_peers/cosmos).
   - Update `~/.gaia/config/config.toml` toml file with seed and persistent peer information from the polkachu.

       ```toml
       # ~/.gaia/config/config.toml
       seeds = "1fa4f7c1d32aa695a0d6c83b5960421e6b2bc981@65.21.33.236:26656, .....,c843429ef1b4a551dab64901991c41af2bc4454f@65.108.130.171:26656"

       persistent_peers = "f4ae01e628935aeb72c52018bd9f9e9e4c8706ee@49.12.165.122:26104,914f2b9c7ee4a6a806ff24cd58379418aed0c121@3.144.11.250:26656,97da81404be96002eb999572ed51f96f460c0b3a@51.91.74.8:26656,67685d93f2256caa7a2d53e3a104f9e437c3d247@95.216.114.244:26656,8220e8029929413afff48dccc6a263e9ac0c3e5e@204.16.247.237:26656"
       ```

5. **Run a Postgres instance and create tables for storing block and transaction data**

   - For the node to support the Postgres database, we will have to set up the PostgreSQL database server and install the Postgres schema.

   - Postgres is available in all the Ubuntu versions. If you are using another OS then follow [this guide](https://www.postgresql.org/download/) to install Postgres.

   - The database schema is provided in [internal/state/indexer/sink/psql/schema.sql](https://github.com/cometbft/cometbft/blob/main/internal/state/indexer/sink/psql/schema.sql#L21-L22) directory. Download this script file locally.
  
   - Now connect to the database and execute the downloaded `schema.sql` script which will create the required Postgres schema. Run the below command to execute the `schema.sql` file, it will prompt you to enter the password. Enter your password and press enter.

       ```psql
       # psql -U <username> -W -d <database-name> -p <port> -h <host> -f <schema.sql file-directory>
       psql -U postgres -W -d cosmoshub -p 5432 -h localhost -f ./schema.sql
       ```
   By now, we have our database named `cosmoshub` running on port 5432.

6. **Configure config.toml for indexing, using psql**

   - Open the `~/.gaia/config/config.toml` file again and modify the indexer field within the `[tx-index]` section to use `psql`.
   - Update the `psql-conn` field with the PostgreSQL configuration. The format for the PostgreSQL connection is `postgresql://<user>:<password>@<host>:<port>/<db>?<opts>`. The PostgreSQL configuration will be `postgresql://postgres:<your-password>@127.0.0.1:5432/cosmoshub`.

       ```toml
       [tx_index]
       indexer = "psql"

       # The PostgreSQL connection configuration, the connection format:
       #  postgresql://<user>:<password>@<host>:<port>/<db>?<opts>
       psql-conn = "postgresql://postgres:<your-password>@127.0.0.1:5432/cosmoshub"
       ```

7. **Start the node and sync blocks using pruned snapshot**

   - We are going to use the pruned snapshot to sync the blocks. Download the pruned snapshot of cosmoshub from [quicksync](https://quicksync.io/networks/cosmos.html).
       ```
       wget https://dl2.quicksync.io/cosmoshub-4-pruned.20240118.0310.tar.lz4
       ``` 
   - Install `lz4` which is required to extract the snapshot.

       ```
       sudo apt update
       sudo apt install snapd -y
       sudo snap install lz4
       ```

   - Follow the below commands to extract the snapshot file in `~/.gaia` directory.

       ```
       lz4 -c -d cosmoshub-4-pruned.20240118.0310.tar.lz4  | tar -x -C $HOME/.gaia
       ```

   At this point, we have successfully installed the Gaia binary, initialized the chain, updated peers and indexing configuration, updated the genesis file with Cosmoshub genesis file, and saved a snapshot file of the CosmosHub for syncing to the chain. It is now time to start the node and be a part of the chain.

8. **Start the node**

   - Now start the node

       ```
       gaiad start
       ```

       It will start syncing blocks and you can see your Postgres table populating.

   Let's retrieve all the transaction data for a specific block height. It is advisable to wait for the blocks to synchronize. To determine the sync status, check the latest block height on [Mintscan](https://www.mintscan.io/cosmos/block) and compare it with the block height indicated in the terminal logs.

9. **Query the transactions for a particular block height**

   - The below query will fetch all the transaction results for block height 18761861(update the height).
   Open a terminal, connect to the PostgreSQL database using psql and run the below query.

   ```sql
   postgres=# SELECT
      tx_results.rowid AS rowid,
      tx_results.block_id AS block_id,
      tx_results.index AS index,
      tx_results.created_at AS created_at,
      tx_results.tx_hash,
      tx_results.tx_result
      FROM blocks
      JOIN tx_results ON blocks.rowid = tx_results.block_id
      WHERE blocks.height=18761861;
   ```
   You can see all the transaction information (something similar to the below table) from the block height 18761861.
  
    ```sql
    rowid   | block_id | index | created_at                    | tx_hash    |       tx_result
    --------|----------|-------|-------------------------------|------------|---------------------------
    1280743 |    49676 |     1 | 2024-01-17 15:30:20.035306+00 | 268C..04F0 | \x088591f9...163635f736571
    1280744 |    49676 |     2 | 2024-01-17 15:30:20.035306+00 | 268C..04F1 | \x088591f9...163635f736571
    1280745 |    49676 |     3 | 2024-01-17 15:30:20.035306+00 | 268C..04F2 | \x088591f9...163635f736571
    1280746 |    49676 |     4 | 2024-01-17 15:30:20.035306+00 | 268C..04F3 | \x088591f9...163635f736571
    ```