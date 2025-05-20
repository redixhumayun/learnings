## Building SlateDB
* Run command from the CLI -> `CLOUD_PROVIDER=local LOCAL_PATH=/tmp/slatedb cargo run --bin slatedb --features cli -- --path /tmp/slatedb` followed by either `list-manifests` or `read-manifest`. However, this doesn't seem to do anything because with a fresh start there are no manifests

## SlateDB Learnings
* Three different components - readers, writers & compactors. These components can run as different processes on different machines.
* The core of the architecture is based on the idea of the manifest ([RFC link here](https://github.com/slatedb/slatedb/blob/main/rfcs/0001-manifest.md#manifest-design)). It seems like the major components all synchronize via the manifest file.
* The manifest has different sections - one for readers, one for writers & one for compactors.
* The manifest is updated via atomic CAS or two-phase writes on object stores.
* Try to pick up [this issue](https://github.com/slatedb/slatedb/issues/288).

## Thread-per-core, IO_Uring & DST Learnings
* Went through [Pekka's repo](https://github.com/redixhumayun/hiisi). It basically models an async i/o interface in `generic.rs`. I still need to understand what `simulation.rs` is doing. It definitely isn't listening to the underlying OS for async i/o but I don't understand what simulation it is doing. Something around the `xmit_queue` is not fitting in my head.
* it seems like DST requires a single-threaded runtime with an IO interface being used anytime network or disk is accessed. In tests, the mocked implementation is used.


## What Is The Goal
* understand the slatedb codebase better because I want to contribute to this.
  * build the project locally
  * do some sample operations and trace the code
  * write down learnings in a markdown file
* understand SSI better. What does this mean? Does it mean write a blog post about it, does it mean I should be able to do an implementation about it, or does it mean something else? I should be able to sketch out what SSI would look like in Notability and have a rough idea of what the algorithm would do. I will timebox this to ~4 hours.
  * understand the slatedb RFC for txn's better
  * go through Phil's post on txn's again.
  * possibly leaf through the papers on SSI - SSI in PostgreSQL and A Critique Of Snapshot Isolation
  * write down what I've understood in Notability
* understand thread-per-core design better, especially around io_uring along with understanding how DST can be implemented because of single threaded design.
  * go through Tonbo's post
  * go through Debasish's tweet he shared on Slack
  * go through codebase of Limbo to understand scheduler & DST better
  * go through hiisi codebase to understand how this implements DST (https://github.com/penberg/hiisi)
  * go through Phil's post about DST (https://notes.eatonphil.com/2024-08-20-deterministic-simulation-testing.html)
  * go through Miller's [Discord message](https://discord.com/channels/852998104931631115/937789984545599528/1208109240850325634)
  * go through the [madsim project](https://github.com/madsim-rs/madsim?tab=readme-ov-file)
  * write a one-page doc in markdown outlining how I can apply this to TLB to implement something for the hackathon
* go through reminders


## SimpleDB

1. The current issue I'm facing is that the test `test planner_tests::test_planner_index_updates` fails because of some lock acquisition issue. For some reason more than one test is competing on the same locks
This was because the same table name and index name was being used across tests. Since the file name ends up being the same and the lock is identified by `BlockId` which consists of file name and block number, there is a significant chance of overlap.