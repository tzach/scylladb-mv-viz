# ScyllaDB Materialized Views & Secondary Indexes

An interactive, single-file visualization of the four ways ScyllaDB can answer the same
question — *"give me the rows for a column that isn't the partition key"* — built as a
big-screen companion to the Advanced Data Modeling deck.

Every mode is shown on three surfaces at once:

- **the CQL** that defines it, with the statement currently executing highlighted,
- **the base table and the table derived from it**, so the reordered primary key is visible
  rather than described,
- **the cluster**, where a write and a read animate hop by hop across coordinator, base
  replicas, and view replicas.

**Disclaimer:** An independent, educational visualization — not an official ScyllaDB product
and not behaviorally exact. Not affiliated with or endorsed by ScyllaDB, Inc.

## The four modes

| Mode | Derived table | Write path | Read path |
| --- | --- | --- | --- |
| **Materialized View** | `heartrate_by_owner`, `PRIMARY KEY (owner, pet_chip_id, time)`, all columns carried | coordinator → base replicas → paired view replicas | one hop to a view replica, full rows come back |
| **Global Index** | `pet_by_owner_index`, `PRIMARY KEY (owner, idx_token, pet_chip_id, time)`, keys only | same as MV | index lookup, **then** a second round-trip to the base partitions |
| **Local Index** | `hr_local_index`, `PRIMARY KEY (pet_chip_id, heart_rate, time)` | view update never leaves the node — base and view replicas coincide | one replica answers the whole query |
| **ALLOW FILTERING** | none | nothing at all | every node scans every partition |

Watching the same INSERT under Local Index and then Global Index is the clearest way to see
what "local" buys: the same three nodes light up as `BASE + VIEW`, and step ③ becomes a loop
inside each node instead of a network hop.

## Fidelity notes

The derived schemas follow ScyllaDB's own construction rather than being invented:

- `index/secondary_index_manager.cc::create_view_for_index()` — a global index puts the
  indexed column in the view's partition key and prepends a computed `idx_token` clustering
  column; a local index keeps the base partition key and puts the indexed column first in the
  clustering key. That's why the index rows in Global Index mode sort by token, not by chip.
- `db/view/view.cc` — each base replica ships its view update to one specific *paired* view
  replica, which is why the write path fans out and then pairs up rather than broadcasting.

Simplifications: 6 nodes, RF=3, CL=ONE, no failures, no repair or view building, a stand-in
hash where murmur3 would be, and short readable keys instead of UUIDs.

## Using it

Static and dependency-free — open `index.html` in a browser. No build step, no server.

| Key | Action |
| --- | --- |
| `1` `2` `3` `4` | switch mode |
| `I` | run the write path (INSERT) |
| `S` | run the read path (SELECT) |
| `R` | reset the data |

The **Speed** slider covers 0.25×–4×; slow it down to narrate a step, speed it up to move on.
The **Theme** button cycles system / light / dark.

## Credits

Styled with the ScyllaDB design system (`scylladb-ds.css`), shared with
[scylladb-ha-demo](https://github.com/tzach/scylladb-ha-demo).
