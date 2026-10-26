# MemIAVL

`Memiavl` is a drop-in replacement for the current iavl implementation, offering a huge performance boost ([benchmark](https://github.com/crypto-org-chain/cronos/wiki/MemIAVL-Benchmark)) for the node. MemIAVL can be enabled by turning on the `memiavl.enable` config item in the `app.toml`. It uses a standalone db directory `data/memiavl.db`. You can always disable it again, in that case the node uses the default iavl `data/application.db` again, so it's ok to switch between them back and forth. Note that `application.db` isn't updated while memiavl is enabled: after switching back, the node resumes from the height `application.db` was left at and re-executes the blocks since then, which requires those blocks to still be in the block store.\
\
`Memiavl` only supports pruned nodes. It ignores the SDK `pruning` settings: the retained history is controlled by `memiavl.snapshot-keep-recent` and `memiavl.snapshot-interval` instead. With the default configuration (`memiavl.snapshot-keep-recent = 1`), the node keeps the latest snapshot plus one older snapshot, so it can serve queries for roughly the last 1000–2000 blocks. This short history is needed for IBC relayers to work.\
\
`Memiavl` can answer gRPC queries at older heights by itself, as long as the height is still covered by a retained snapshot. In order to support historical grpc queries beyond that window, you should enable versiondb together with memiavl. If you need to support archive merkle proof generations, don't use `memiavl`!\
\
The default `memiavl` section in `app.toml`:

```toml
[memiavl]

# Enable defines if the memiavl should be enabled.
enable = false

# ZeroCopy defines if the memiavl should return slices pointing to mmap-ed buffers directly (zero-copy),
# the zero-copied slices must not be retained beyond current block's execution.
# the sdk address cache will be disabled if zero-copy is enabled.
# only affects latest-height reads; historical-height queries always copy, since their snapshot can be
# unmapped by the historical store cache while a response is still in flight.
zero-copy = false

# AsyncCommitBuffer defines the size of asynchronous commit queue, this greatly improve block catching-up
# performance, -1 means synchronous commit.
async-commit-buffer = 0

# SnapshotKeepRecent defines what many old snapshots (excluding the latest one) to keep after new snapshots are
# taken, defaults to 1 to make sure ibc relayers work.
snapshot-keep-recent = 1

# SnapshotInterval defines the block interval the memiavl snapshot is taken, default to 1000.
snapshot-interval = 1000

# CacheSize defines the size of the cache for each memiavl store, default to 1000.
cache-size = 1000
```

### Configuration Notes

* `async-commit-buffer`: `-1` means synchronous commit. `0` (the default) already enables asynchronous commit with an unbuffered queue; a positive value adds a buffer of that size.
* `zero-copy`: enabling it also disables the SDK address cache, because cached zero-copied slices would be unsafe. It only affects reads at the latest height; historical-height queries always copy.
* `cache-size`: Cronos only passes it to memiavl when both of these hold; otherwise the cache size is `0`, which disables the cache:
* `block-executor` is not `block-stm`, and
* `cronos.disable-optimistic-execution = true`.

Optimistic execution is enabled by default, so on a default node `cache-size` has no effect and the cache is disabled.

{% hint style="danger" %}
Don't run `block-executor = "sequential"` with `disable-optimistic-execution = true` and a non-zero `cache-size` on v1.8. In that combination the cache can keep a stale value when a CheckTx (under `mempool.type = "app"`) or a latest-height query reads a key while the block commit writes it, and the node then halts with an app hash mismatch. If you need that executor combination, set `cache-size = 0`. See the [v1.7 to v1.8 migration guide](https://github.com/crypto-org-chain/cronos/blob/v1.8.0/docs/migrations/v1.7-to-v1.8.md#62-do-not-enable-the-memiavl-node-cache).\
<br>
{% endhint %}

*   `snapshot-writer-limit` (optional, not in the generated `app.toml`): the maximum number of concurrent workers used to write a memiavl snapshot. Defaults to `4`. To change it, add it to the `[memiavl]` section:

    ```toml
    snapshot-writer-limit = 4
    ```

### Recommended Configuration

The [v1.7 to v1.8 migration guide](https://github.com/crypto-org-chain/cronos/blob/v1.8.0/docs/migrations/v1.7-to-v1.8.md#6-block-stm-and-memiavl) recommends the same memiavl settings for validators, RPC and archive nodes:

```toml
# app.toml
[memiavl]
enable = true
zero-copy = true
```

Keep Block-STM (`[evm] block-executor = "block-stm"`) or optimistic execution (`[cronos] disable-optimistic-execution = false`) on, so that `cache-size` stays disabled (see the warning above).

`zero-copy = true` costs nothing when Block-STM is on. Its only side effect is turning off the SDK address cache, and Block-STM already turns that cache off.

### Database Backend

memiavl stores the application state in its own file format under `data/memiavl.db`, so it doesn't depend on any database backend setting:

* `db_backend` in `config.toml` selects the backend for CometBFT's databases (block store, state, tx index).
* `app-db-backend` in `app.toml` selects the backend for `data/application.db` and the state-sync snapshot metadata (`data/snapshots/metadata.db`). With memiavl enabled, the node doesn't keep its state in `application.db`.

So memiavl works with goleveldb or RocksDB. In practice, use RocksDB:

* The memiavl snapshots on [snapshot.cronos.com](https://snapshot.cronos.com/) are only published for RocksDB (`mainnet-snapshot/rocksdb/memiavl/` and `mainnet-snapshot/versiondb/memiavl/`).
* versiondb requires RocksDB.

{% hint style="info" %}
`[rocksdb] node_type` only tunes how `data/application.db` is opened. A memiavl node doesn't read its state from `application.db`, so `node_type` has no effect on state reads there. It doesn't apply to versiondb either.
{% endhint %}

### Use Cases

**Semi-Archived Node**\
When versiondb was released, we recommended users to set up a pruned iavl tree together with versiondb to run a semi-archived node. You can replace the pruned iavl tree with `memiavl` now.\
\
**State Sync Node**\
`Memiavl` can do state-sync snapshot restoration much faster than the current iavl, it's actually much faster than the chunk downloading speed, with `memiavl`, allowing state-sync node to be bootstrapped in around 10 minutes depending on the internet speed (measured when memiavl was released; it grows with the state size). If you download a snapshot from CDN and do a local restoration, it can even be faster. Just enable memiavl in `app.toml` before starting a state-sync.\
\
**Snapshot Providers**\
`memiavl` can do state-sync snapshot exports much faster as well, on Cronos mainnet, snapshots could be exported in minutes instead of days when memiavl was released, so it's recommended to run snapshot provider nodes with `memiavl`, so the snapshots will be much more up-to-date.\
\
A memiavl node can only export a state-sync snapshot at a height where memiavl itself has taken a snapshot, and the SDK only takes a state-sync snapshot at multiples of `[state-sync] snapshot-interval` in `app.toml`. So state-sync snapshots are only produced at heights that are multiples of both intervals. Set `[state-sync] snapshot-interval` to a multiple of `memiavl.snapshot-interval`:

```toml
# app.toml
[memiavl]
snapshot-interval = 1000

[state-sync]
snapshot-interval = 1000   # or 2000, 3000, ...; a value like 1500 only yields snapshots every 3000 blocks
```

The default `[state-sync] snapshot-interval` is `0`, which disables state-sync snapshots.

### Running With VersionDB

When `versiondb` is enabled, it serves all gRPC and JSON-RPC state queries, including latest-height queries and `eth_call`; memiavl is then only used for block execution. So on such a node, memiavl's `zero-copy` and `cache-size` settings don't affect query speed.

### Rollback

`cronosd rollback` works with memiavl. It rolls the CometBFT state and the application state back by one height, and memiavl discards its history after that height. The target height must still be covered by a retained memiavl snapshot; otherwise the rollback fails with `target version is pruned`. A single rollback normally stays inside that window, but repeated rollbacks can run past it.

### Migrate Semi-Archive Node

To migrate a semi-archive versionDB node to memiavl, you need a memiavl db at the same height as the versiondb, then continue normal syncing from there. Restore a state-sync snapshot from below the versiondb height with the local state-sync commands, then sync memiavl up to the versiondb height:

```toml
# download a snapshot whose version is smaller than versiondb's latest version
$ cronosd snapshots load /path/to/downloaded-snapshot.tar.gz

# check the snapshot height and format in list command
$ cronosd snapshots list

# edit app.toml to enable memiavl, then restore using the height and format
# shown by `snapshots list` (the current snapshot format is 3)
$ cronosd snapshots restore <snapshot height> <snapshot format>

# edit app.toml to disable versiondb, and set `memiavl.async-commit-buffer = -1`.
# --halt-height H stops the node before executing block H, so pass versiondb height + 1
# to stop with memiavl exactly at the versiondb height.
$ cronosd start --halt-height <versiondb height + 1>

# edit app.toml to enable versiondb, and set `memiavl.async-commit-buffer = 0 or small positive number` to re-enable async commit.
$ cronosd start
```

{% hint style="info" %}
Stop with memiavl exactly at the versiondb height, so that memiavl and versiondb are at the same height when versiondb is re-enabled. Blocks executed while versiondb is disabled are not recorded in versiondb. If memiavl stops one block short, the node simply executes that block after versiondb is re-enabled. If memiavl does run past that height, the node rolls memiavl back to the versiondb height when versiondb is re-enabled, and re-executes the blocks after it. This only works while that height is still covered by a retained memiavl snapshot; otherwise memiavl fails to load with `target version is pruned`.
{% endhint %}

### Compression

Right now memiavl doesn't do any generic compressions to the data files, that would kill the simplicity of the current implementation, and probably hurt performance if not done right. Fortunately, on the linux filesystem level compressions work well with `mmap`. We ran memiavl on `btrfs` configured with `zstd` compression, and observed around \\\~60% compression rate on the memiavl directory, with no visible performance regression (measured when memiavl was released; results depend on the data). So this is the recommended way to run memiavl for efficient disk space usage.

### Disk Usage

* Snapshot rewrites: a new snapshot is written in full before older ones are pruned. Plan for (`snapshot-keep-recent` + 2) snapshots plus the write-ahead log at peak, e.g. 3 snapshots with the default `snapshot-keep-recent = 1`.
* `data/application.db`: once the node runs on memiavl, `application.db` is no longer updated, and the node doesn't read state from it. You can delete it to reclaim space, but only if you won't switch back to IAVL. Switching back would then require restoring `application.db` from a snapshot.

### Troubleshooting

| <p>Error<br></p>                                                                 | <p>Cause<br></p>                                                                                                                                                                                                                                                                         | <p>Fix<br></p>                                                                                                                                                                 |
| -------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| <p><code>target version is pruned</code><br></p>                                 | <p>The node needs a height older than the oldest retained memiavl snapshot, e.g. a rollback, or versiondb re-enabled after memiavl ran past it.<br></p>                                                                                                                                  | <p>Restore memiavl from a snapshot at or below the needed height. Raise <code>snapshot-keep-recent</code> if you need a longer window.<br></p>                                 |
| <p><code>fail to lock db: file already locked</code><br></p>                     | <p>Another process (for example a second <code>cronosd</code>) has <code>data/memiavl.db</code> open.<br></p>                                                                                                                                                                            | <p>Stop the other process.<br></p>                                                                                                                                             |
| <p><code>panic: mempool.type=app is incompatible with NoOpMempool</code><br></p> | <p><code>mempool.type = "app"</code> in <code>config.toml</code> while <code>app.toml</code> has <code>[mempool] max-txs = -1</code> (or a negative <code>feebump</code>), which turns the app mempool off. Not memiavl-specific, but often hit when applying the v1.8 settings.<br></p> | <p>Set <code>[mempool] max-txs</code> in <code>app.toml</code> to a positive value, e.g. <code>10000</code>, and keep <code>feebump</code> at <code>0</code> or above.<br></p> |
| <p>App hash mismatch after enabling <code>cache-size</code><br></p>              | <p>The memiavl node cache was active (<code>block-executor = "sequential"</code> with <code>disable-optimistic-execution = true</code>); see the warning under Configuration Notes.<br></p>                                                                                              | <p>Set <code>cache-size = 0</code>, or keep Block-STM or optimistic execution on.<br></p>                                                                                      |
