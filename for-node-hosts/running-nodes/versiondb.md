# VersionDB

### Overview

Anyone who has been running archive nodes for a while is familiar with the ever-growing disk size of their nodes. For Cronos, it's the IAVL database of archive nodes, `application.db`, that keeps growing. Migrating to VersionDB dramatically reduces disk pressure while maintaining fast and reliable access to historical state.\
\
VersionDB stores multiple versions of on-chain state key-value pairs without using a merkelized tree structure like the IAVL tree, making both DB size and query performance much better than IAVL trees. However, VersionDB does not perform root hash and Merkle proof generation, so for those features we still need the IAVL tree. gRPC queries do not need to support proof generation, so VersionDB alone is enough to serve them. You can also start a standalone gRPC query service with the `--grpc-only` flag of `cronosd start`.\
\
After migrating to VersionDB, it's safe to prune the IAVL tree to reclaim substantial disk space, as long as you don't need to generate Merkle proofs on historical versions. The pruned tree can be either a pruned IAVL `application.db`, or \[MemIAVL]\(memiavl.md) (experimental).

### Requirements

VersionDB is built on RocksDB, so it is only available in `cronosd` binaries compiled with the `rocksdb` build tag:

* The official release binaries include it.
* If you build from source, build with `COSMOS_BUILD_OPTIONS=rocksdb make build` (a RocksDB library must be installed), or use the Nix build.

With a binary built without it, enabling versiondb fails at startup with `versiondb is not supported in this binary`, and the `cronosd changeset` command is not available.

### Limitations

The limitations of the setup with VersionDB and a pruned IAVL tree are:

* Currently, this solution is only recommended for archive and non-validator nodes to try (validator nodes are recommended to do pruning).
* Different implementations exist for VersionDB. Our current implementation is based on RocksDB's user-defined timestamp feature, and stores the data in a standalone RocksDB instance. It does not support other db backends yet. The other databases in the node still support multiple backends as before.
* Does not support `eth_getProof` and non-gRPC `abci_query` for the historical versions that are pruned in the IAVL tree. The other APIs function just like an archive node as before.

### Config

The `versiondb` section in `app.toml`:

```
[versiondb]
# Enable defines if the versiondb should be enabled.
enable = true
```

The db instance is placed at the `$NODE_HOME/data/versiondb` directory. Currently this path cannot be customized. Enabling it switches the gRPC query service's backing store from the IAVL tree to VersionDB.

{% hint style="warning" %}
When versiondb is enabled and not empty, the node loads the IAVL tree (or MemIAVL) at the versiondb's latest version instead of the tree's own latest version, so that versiondb never ends up with a gap:

* If the tree is ahead of versiondb, the node resumes from the versiondb height, and CometBFT re-executes the later blocks from its block store. See \[Catch up with the IAVL tree]\(#catch-up-with-the-iavl-tree) to avoid a long re-execution.
* If the tree is behind versiondb, an IAVL `application.db` fails to load, since it has no state at the versiondb height. MemIAVL loads its own latest version instead.
{% endhint %}

### Tutorial 1 - Migrating from snapshot

#### Step 1.  Extract from snapshot

Download a versiondb snapshot from the official [Cronos snapshots](https://snapshot.cronos.com/) (see \[Cronos Native Snapshots]\(../cronos-evm-snapshots/cronos-native-snapshots.md)), under `mainnet-snapshot/versiondb/`. Several variants are offered (`archive`, `pruned`, `default`, `memiavl`); pick the one matching the tree you want to run next to versiondb.

Extract it so that the versiondb data ends up in your `$NODE_HOME/data/versiondb` folder, and the tree (`application.db` or `memiavl.db`) next to it in `$NODE_HOME/data/`. The snapshots contain a top-level `data/` folder, so extract them into `$NODE_HOME`:

```bash
# single-file snapshot
$ lz4 -dc <snapshot>.tar.lz4 | tar -xf - -C $NODE_HOME

# large snapshots are split into .partNNN files; concatenate them in order while decompressing
$ cat <snapshot>.tar.lz4.part* | lz4 -dc | tar -xf - -C $NODE_HOME
```

To check the layout before extracting, list the first entries with `cat <snapshot>.tar.lz4* | lz4 -dc | tar -tf - | head`.

#### Step 2. Update config

Enable versiondb in `app.toml` as shown in \[Config]\(#config). The `application.db` in these snapshots is a RocksDB database (it's made of `.sst` files), so also set `app-db-backend = "rocksdb"`.

#### Step 3. Start the node

Restart the node. If the `application.db` has no IAVL fast node index, the node builds it on startup, unless `iavl-disable-fastnode = true` is set in `app.toml`.

### Tutorial 2 - Bootstrap from a state-sync snapshot

If you don't need history from before a certain height, you can build an initial versiondb directly from a state-sync snapshot, without an archive node. The versiondb then holds history from the snapshot height onwards.

```
# load a downloaded state-sync snapshot into the local snapshot store
$ cronosd snapshots load /path/to/downloaded-snapshot.tar.gz

# check the snapshot height and format (the current snapshot format is 3)
$ cronosd snapshots list

# build $NODE_HOME/data/versiondb from the snapshot
$ cronosd changeset restore-versiondb <snapshot height> <snapshot format>

# restore the pruned tree at the same height, e.g. with memiavl enabled in app.toml
$ cronosd snapshots restore <snapshot height> <snapshot format>

# on a new node with an empty block store only: bootstrap the CometBFT state at the snapshot height
$ cronosd comet bootstrap-state --height <snapshot height>
```

`comet bootstrap-state` verifies the state with a light client, so it needs the `[statesync]` section of `config.toml` filled in (`rpc_servers`, `trust_height`, `trust_hash`, `trust_period`). It refuses to run if the block store isn't empty; skip it when restoring on a node that already has blocks.\
\
Then enable versiondb in `app.toml` (see \[Config]\(#config)) and start the node.

### Tutorial 3 - Starting from scratch

This tutorial starts from an existing archive node, so prepare enough time to go through this migration, especially the change set extraction, which may take days to complete, even with an `r6g.16xlarge` instance. If you want to skip this, use the snapshot or state-sync tutorials above. For more information on the different steps, see the documentation [here](https://github.com/crypto-org-chain/cronos/wiki/VersionDB-Migration).

We will be following the following workflow:

1. Extract state change sets from the existing archived IAVL tree.
2. Build the versiondb from the change set files.
3. Build a clean application.db from the change set files.

<figure><img src="https://1786307500-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FqXsIo4b4WtOTfICyOIxa%2Fuploads%2FwC7TGg4jrO6dEr75XQDF%2Fversiondb2.png?alt=media&#x26;token=df15f3cc-0467-4747-b5c2-ee21aa35589c" alt=""><figcaption></figcaption></figure>

The commands below use these paths; adjust them to your setup:

```bash
export NODE_HOME=/chain/.cronosd
export CRONOSD=$NODE_HOME/cosmovisor/current/bin/cronosd
export CHANGESET_DIR=/s3-upload/changeset
export SST_DIR=/s3-upload/sst
export WORK_DIR=/s3-upload
```

#### Step 0. Prepare

* Update your binary (built with RocksDB support, see \[Requirements]\(#requirements)) and stop cronosd.

#### Step 1. Extract change sets

* Get the end version before extracting change sets.

```bash
ulimit -n 60000 # python-iavl and cronosd changeset subcommand open many files
export MAXIMUM_VERSION=$(nix run github:crypto-com/python-iavl/main -- commit-infos --db $NODE_HOME/data/application.db/ | head -n 1 | awk '{ print $2 }')
export END_VERSION=$((MAXIMUM_VERSION + 1)) # --end-version is exclusive
```

* Extract Change Sets

This step takes the longest. The migration process will try to parallelize the tasks as much as possible, and hence will use significant RAM. There are flags to control the concurrency level and RAM usage for different machine specs.

Time taken: around 4 days (512G RAM, r6g.16xlarge) for a mainnet archive node, as measured when versiondb was released in 2023. Expect it to take longer now that the chain is bigger.

```bash
$ $CRONOSD changeset dump $CHANGESET_DIR \
  --home $NODE_HOME --end-version $END_VERSION --concurrency 128 --iavl-version 0
```

{% hint style="warning" %}
`--iavl-version` tells the dump which IAVL node format to use when finding where each store's history begins. It defaults to `1`. For an archive node whose history goes back to before the IAVL v1 upgrade, pass `--iavl-version 0`. Otherwise the dump only covers the versions written after the upgrade, or skips stores entirely. Use `--iavl-version 1` only for a node whose whole history was written by IAVL v1.
{% endhint %}

* Verify Change Sets

Time taken: around 1 hour

```bash
#!/bin/bash
set -e
set +o pipefail

# expected per-store hashes, from the commit info stored in application.db at the latest version
COMMIT_INFOS=$(nix run github:crypto-com/python-iavl/main -- commit-infos --db $NODE_HOME/data/application.db/)
stores=($($CRONOSD changeset default-stores --home $NODE_HOME))
for i in "${stores[@]}"
do
  expected=$(echo "$COMMIT_INFOS" | grep "store name: $i," | awk -F'hash: ' '{print $2}' | sed 's/^0x//')
  actual=$($CRONOSD changeset verify $CHANGESET_DIR --stores $i | tail -1 | jq -r ".storeInfos[] | select(.name == \"$i\") | .commitId.hash" | base64 -d | xxd -p -c 32)
  if [[ -n "$expected" && "$expected" == "$actual" ]]; then
      echo "$i store is verified"
  else
      echo "$i store has issue"
  fi
done
```

{% hint style="info" %}
The script compares against the commit info that the node stored at the latest version, which works whatever the IAVL node format is. Don't use `python-iavl root-hash` for this: it only reads legacy IAVL v0 root keys, so it prints an empty hash for versions written after the IAVL v1 upgrade.
{% endhint %}

```
./verify.sh
```

#### Step 2. Build VersionDB

This is a two-phase process for efficiently constructing a version database from `changeset` data.

The first step is to generate SST files from changeset data:

{% hint style="info" %}
SST files are a binary format used by embedded databases (like RocksDB/LevelDB). They store key-value pairs in sorted order, making them efficient for bulk ingestion. Rather than inserting data one record at a time, batching into SSTs is dramatically faster.
{% endhint %}

```bash
$ $CRONOSD changeset build-versiondb-sst $CHANGESET_DIR $SST_DIR
```

Then create the VersionDB directly in the node's data directory:

```bash
$ $CRONOSD changeset ingest-versiondb-sst \
  $NODE_HOME/data/versiondb $SST_DIR/*.sst --move-files --maximum-version $MAXIMUM_VERSION
```

{% hint style="info" %}
This process took up to 1 hour on a mainnet archive node.`application.db` doesn't contain the fast node index; it's rebuilt automatically when the node starts, unless `iavl-disable-fastnode = true` is set in `app.toml`.
{% endhint %}

{% hint style="warning" %}
`restore-app-db` only generates a RocksDB `application.db`. Make sure `app-db-backend = "rocksdb"` is set in `app.toml` before starting the node.
{% endhint %}

#### Step 3. Restore IAVL Tree

Rebuild a clean IAVL tree at the latest version from the change sets. The restored&#x20;

```bash
# replay the change sets into a memiavl-format snapshot of the latest version
$ $CRONOSD changeset verify $CHANGESET_DIR --save-snapshot $WORK_DIR/snapshot
# build application.db from that snapshot
$ $CRONOSD changeset restore-app-db $WORK_DIR/snapshot $WORK_DIR/application.db
# replace the archive application.db with the new one
$ mv $NODE_HOME/data/application.db $WORK_DIR/application.db.archive
$ mv $WORK_DIR/application.db $NODE_HOME/data/application.db
```

Once the node is confirmed working, `$WORK_DIR/application.db.archive` can be deleted.

{% hint style="info" %}
This process took up to a few hours on a mainnet archive node.
{% endhint %}

#### Step 4. Update config

Enable versiondb in `app.toml` as shown in \[Config]\(#config), and make sure `app-db-backend = "rocksdb"`.

#### Step 5. Start the node

Restart the node. It builds the IAVL fast node index on startup (see Step 3).

### Catch up with the IAVL tree

If an existing versiondb lags behind the IAVL tree, the node resumes from the versiondb height and re-executes every later block on startup (see \[Config]\(#config)). If the gap is large, or those blocks are no longer in the block store, catch versiondb up offline instead: dump the missing block range, write it into the existing versiondb, then update its latest version. Stop the node first.\
\
This requires an IAVL `application.db` that still contains every version in the range, so the node's `pruning` settings must not have removed them yet. With MemIAVL, use `changeset dump-memiavl` with the same `--start-version`/`--end-version` flags instead; it only covers the versions MemIAVL still retains.

{% hint style="warning" %}
Don't use the SST commands (`build-versiondb-sst` / `ingest-versiondb-sst` with files) on an existing versiondb: RocksDB rejects ingesting files whose key ranges overlap data already in the db. Use `to-versiondb` as below.
{% endhint %}

```bash
export CATCHUP_CHANGESET_DIR=/s3-upload/changeset-catchup

# --start-version is inclusive, --end-version is exclusive.
# keep the default --iavl-version 1 here: the catch-up range is written by IAVL v1, and
# --iavl-version 0 skips every store once the legacy v0 roots have been pruned.
$ $CRONOSD changeset dump $CATCHUP_CHANGESET_DIR --home $NODE_HOME \
  --start-version <versiondb latest version + 1> --end-version <iavl latest version + 1>

# write each store's change sets into the existing versiondb
$ for store in $($CRONOSD changeset default-stores --home $NODE_HOME); do
    $CRONOSD changeset to-versiondb $NODE_HOME/data/versiondb $CATCHUP_CHANGESET_DIR/$store/* --store $store
  done

# to-versiondb doesn't update the latest version; set it (no sst files needed)
$ $CRONOSD changeset ingest-versiondb-sst $NODE_HOME/data/versiondb --maximum-version <iavl latest version>
```

### Other `changeset` commands

| <p>Command<br></p>                                                                                | <p>Purpose<br></p>                                                                                                                                                                                  |
| ------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| <p><code>dump-memiavl &#x3C;outDir></code><br></p>                                                | <p>Extract change sets from <code>data/memiavl.db</code> instead of <code>application.db</code>.<br></p>                                                                                            |
| <p><code>print &#x3C;plain-file></code><br></p>                                                   | <p>Pretty-print the content of a change set file.<br></p>                                                                                                                                           |
| <p><code>to-versiondb &#x3C;versiondb-path> &#x3C;plain-1> ... --store &#x3C;name></code><br></p> | <p>Feed one store's change set files into versiondb directly, without the SST step; works on an existing versiondb. <code>--store</code> is required, and the latest version isn't updated.<br></p> |
| <p><code>fixdata &#x3C;dir></code><br></p>                                                        | <p>Fix wrong data in an existing versiondb (see <a href="https://github.com/crypto-org-chain/cronos/issues/1683">#1683</a>); <code>--dry-run</code> opens it read-only and writes nothing.<br></p>  |

Run `cronosd changeset <command> --help` for the full list of flags.

### Results on Cronos

When versiondb was released in 2023, we measured disk size reductions of around \\\~`63%` for a Cronos mainnet archive node. The absolute sizes below are from that time; today's chain is larger. The reduction you see will vary from case to case, but there's a good improvement in disk size to be gained by moving to versionDB.\
\
Rocksdb archive node:

```bash
$ du -hd1 /chain/.cronosd/data/
1.6T /chain/.cronosd/data/application.db
103G /chain/.cronosd/data/blockstore.db
1.1G /chain/.cronosd/data/cs.wal
121M /chain/.cronosd/data/evidence.db
114M /chain/.cronosd/data/snapshots
163G /chain/.cronosd/data/state.db
462G /chain/.cronosd/data/tx_index.db
2.3T /chain/.cronosd/data/
```

Versiondb archive node:

```bash
$ du -hd1 /chain/.cronosd/data/
82G /chain/.cronosd/data/application.db
104G /chain/.cronosd/data/blockstore.db
26G /chain/.cronosd/data/versiondb
1.1G /chain/.cronosd/data/cs.wal
115M /chain/.cronosd/data/evidence.db
112M /chain/.cronosd/data/snapshots
163G /chain/.cronosd/data/state.db
463G /chain/.cronosd/data/tx_index.db
837G /chain/.cronosd/data/
```
