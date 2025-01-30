## Running With strace

I probably ran with `strace -e trace=mmap,munmap,brk -f ./target/debug/deps/hashmap-[hash] profile_memory_patterns`
```shell
Resize Stats:
  Old capacity: 1048576, New capacity: 2097152
  Entry size: 48 bytes
  New vec size: 100663296 bytes
  Current size (items): 734004
[pid  2017] 18:33:19 mmap(NULL, 50335744, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7bc4c0e00000
  Actual old vec size: 50331648 bytes
[pid  2017] 18:33:19 mmap(NULL, 100667392, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7bc49de00000
[pid  2017] 18:33:20 munmap(0x7bc4b8e00000, 50335744) = 0
[pid  2017] 18:33:20 munmap(0x7bc4c0e00000, 50335744) = 0
```

In the above logs, we can see two calls to `mmap` and two calls to `munmap`. It roughly maps to this code

```rust
fn resize(&mut self) {
    let old_capacity = self.capacity;
    let new_capacity = self.capacity * 2;
    
    // Calculate sizes
    let entry_size = std::mem::size_of::<Entry<K, V>>();
    let vec_size = new_capacity * entry_size;
    
    println!("Resize Stats:");
    println!("  Old capacity: {}, New capacity: {}", old_capacity, new_capacity);
    println!("  Entry size: {} bytes", entry_size);
    println!("  New vec size: {} bytes", vec_size);
    println!("  Current size (items): {}", self.size);
    
    // Track existing data
    let old_entries: Vec<Entry<K, V>> = self.data.drain(..).collect();
    println!("  Actual old vec size: {} bytes", old_entries.len() * entry_size);

    //  do the resizing
    self.capacity = new_capacity;
    let mut new_data: Vec<Entry<K, V>> = vec![Entry::Empty; new_capacity];
    for entry in old_entries {
        if let Entry::Occupied(k, v) = entry {
            let mut index = self.hash(&k);
            while let Some(Entry::Occupied(_, _)) = new_data.get(index) {
                index = (index + 1) % self.capacity;
            }
            new_data[index] = Entry::Occupied(k, v);
        }
    }
    self.data = new_data;
    println!("Done resizing!!!");
}
```

So the reason seems to be that the `collect()` is allocating a new vector before deallocating that along with the old `data` at the end of the function. Now, presumably, once the additional allocation is reduced, that should get rid of one `mmap` & `munmap` pair.

And indeed, that is exactly what happens once the code is changed. Changes below

```rust
fn resize(&mut self) {
        let old_capacity = self.capacity;
        let new_capacity = self.capacity * 2;

        // Calculate sizes
        let entry_size = std::mem::size_of::<Entry<K, V>>();
        let vec_size = new_capacity * entry_size;

        println!("Resize Stats:");
        println!(
            "  Old capacity: {}, New capacity: {}",
            old_capacity, new_capacity
        );
        println!("  Entry size: {} bytes", entry_size);
        println!("  New vec size: {} bytes", vec_size);
        println!("  Current size (items): {}", self.size);
        println!(
            "  Actual old vec size: {} bytes",
            self.data.len() * entry_size
        );

        let new_data: Vec<Entry<K, V>> = vec![Entry::Empty; new_capacity];
        let old_data = std::mem::replace(&mut self.data, new_data);
        self.capacity = new_capacity;
        for entry in old_data {
            if let Entry::Occupied(k, v) = entry {
                let mut index = self.hash(&k);
                while let Some(Entry::Occupied(_, _)) = self.data.get(index) {
                    index = (index + 1) % self.capacity;
                }
                self.data[index] = Entry::Occupied(k, v);
            }
        }
        println!("Done resizing!!!");
    }
```

```shell
Resize Stats:
  Old capacity: 262144, New capacity: 524288
  Entry size: 48 bytes
  New vec size: 25165824 bytes
  Current size (items): 183501
  Actual old vec size: 12582912 bytes
[pid  2485] 18:54:54 mmap(NULL, 25169920, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x757a24a00000
[pid  2485] 18:54:54 munmap(0x757a26400000, 12587008) = 0
Done resizing!!
```

Unfortunately, it doesn't seem to be possible to emulate this easily on Mac with `strace`.

## Running With perf
Recording with perf is as simple as `sudo perf record -g ./target/debug/hashmap`. The `sudo` is required to override some permissions on an AWS Ubuntu VM. The output of `perf` can be controlled with `sudo perf record -g -o perf_output.data ./target/debug/hashmap`.

The above generates a file called `perf.data` which can be explored using `perf report`. The output of `perf report` looks like below

```shell
Samples: 10K of event 'task-clock:ppp', Event count (approx.): 2676250000
  Children      Self  Command  Shared Object         Symbol
+   98.71%     0.00%  hashmap  hashmap               [.] _start
+   98.71%     0.00%  hashmap  libc.so.6             [.] __libc_start_main
+   98.70%     0.00%  hashmap  hashmap               [.] std::rt::lang_start
+   19.74%     0.35%  hashmap  libc.so.6             [.] malloc
+   19.16%     0.30%  hashmap  hashmap               [.] alloc::raw_vec::RawVecInner<A>::with_capacity_in
+   18.79%     0.15%  hashmap  hashmap               [.] alloc::raw_vec::RawVecInner<A>::try_allocate_in
+   17.30%     0.00%  hashmap  libc.so.6             [.] 0x00007298416ac50a
+   16.09%     0.00%  hashmap  [kernel.kallsyms]     [k] 0xffffffff9a000bc7
+   16.09%     0.00%  hashmap  [kernel.kallsyms]     [k] 0xffffffff99e096b3
+   10.63%     0.00%  hashmap  [kernel.kallsyms]     [k] 0xffffffff98cd13f7
+   10.30%    10.28%  hashmap  hashmap               [.] <core::hash::sip::Hasher<S> as core::hash::Hasher>::write
+   10.19%     0.00%  hashmap  [kernel.kallsyms]     [k] 0xffffffff9901922c
+    9.76%     0.00%  hashmap  [kernel.kallsyms]     [k] 0xffffffff99018f5e
+    9.59%     0.00%  hashmap  [kernel.kallsyms]     [k] 0xffffffff990188e7
+    9.03%     9.01%  hashmap  hashmap               [.] hashmap::HashMap<K,V>::insert
+    8.88%     8.87%  hashmap  hashmap               [.] core::intrinsics::copy_nonoverlapping::precondition_check
+    8.48%     0.03%  hashmap  hashmap               [.] core::ptr::drop_in_place<alloc::raw_vec::RawVec<u8>>
```

The first percentage is the amount of time spent in the children of this call and the second percentage is the amount of time spent within this function itself. The `[.]` and the `[k]` represent whether this is a user-level or kernel call.

These two lines have to do with allocating capacity for the vector storing the data.

```shell
+   19.16%     0.30%  hashmap  hashmap               [.] alloc::raw_vec::RawVecInner<A>::with_capacity_in
+   18.79%     0.15%  hashmap  hashmap               [.] alloc::raw_vec::RawVecInner<A>::try_allocate_in
```

The line below is Rust's safety check

```shell
+    8.88%     8.87%  hashmap  hashmap               [.] core::intrinsics::copy_nonoverlapping::precondition_check
```

The line below is about dropping (or cleaning up) the memory of the vector

```shell
+    8.48%     0.03%  hashmap  hashmap               [.] core::ptr::drop_in_place<alloc::raw_vec::RawVec<u8>>
```

### Generating A Flamegraph
To generate a flamegraph, do the following

1. Clone the relevant repository with `git clone https://github.com/brendangregg/FlameGraph.git`
2. Add to $PATH  with `export PATH=$PATH:$(pwd)/FlameGraph`
3. Run `sudo perf script -i <perf_data_file> | stackcollapse-perf.pl | flamegraph.pl > flamegraph.svg`
4. Copy the file to local machine (if doing this on a remote server) with `scp -i /path/to/your/key.pem ubuntu@your-ec2-ip:~/hashmap-rs/flamegraph.svg ./`
5. The SVG can be visualised in the browser.

### Running Profiling On A Mac

Use Instruments to generate and visualise data

When trying to run `xcrun xctrace`, an issue like this might pop up
```shell
xcrun xctrace record --template "Time Profiler" --template "Allocations" --launch ./target/debug/hashmap
Starting recording with the Allocations template. Launching process: hashmap.
xctrace: Instruments wants permission to analyze other processes. Please enter an administrator username and password to allow this.
Username (zaidhumayun):
Password:

Ctrl-C to stop the recording
Run issues were detected (trace is still ready to be viewed):
* [Error] Failed to gain authorization

    * [Error] Recovery Suggestion: Target binary needs to be debuggable and signed with 'get-task-allow

Recording failed with errors. Saving output file...
Output file saved as: Launch_hashmap_2024-11-24_22.06.11_1519D5E2.trace
```

To get around this, sign the binary with the following

```shell
cargo build [--debug] # create the debug build

# Create entitlements file
echo '<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.get-task-allow</key>
    <true/>
</dict>
</plist>' > debug.entitlements

# Then sign
codesign -s - --entitlements debug.entitlements --force ./target/debug/hashmap
```

To instrument for allocations -> `xcrun xctrace record --template "Allocations" --launch ./target/debug/hashmap -o ./profile_data`
To instrument for time profiling -> `xcrun xctrace record --template "Time Profiler" --launch ./target/debug/hashmap -o ./profile_data`

## Notes
### Workflow
* Write basic code on Mac OS and use for basic performance measurements using tools like `time` etc.
* Run most profiling on c6i.2xlarge EC2 instance (8vCPU, 16GBRAM) ($0.34/hour)
* Run full profile on c6i.metal EC2 instance (128vCPU, 256GBRAM) ($5.44/hour)

**Cost**
c6i.2xlarge (ap-south-1: $0.336/hour):

8 hours/week = $2.69/week
Monthly = ~$11 (4.1 weeks)

c6i.metal (ap-south-1: $5.376/hour):

2 hours/week = $10.75/week
Monthly = ~$44 (4.1 weeks)

Total monthly: ~$55


### Setting Up Fresh EC2 For Profiling

```shell
#!/bin/bash

# Exit on any error
set -e

echo "Starting EC2 setup for performance profiling..."

# Update system
sudo apt-get update && sudo apt-get upgrade -y

# Install essential build tools and profiling tools
sudo apt-get install -y \
    build-essential \
    curl \
    wget \
    git \
    linux-tools-common \
    linux-tools-generic \
    linux-tools-`uname -r` \
    strace

# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source $HOME/.cargo/env

# Clone FlameGraph repository
cd $HOME
git clone https://github.com/brendangregg/FlameGraph.git
echo "export PATH=\$PATH:\$HOME/FlameGraph" >> $HOME/.bashrc

# Create projects directory
mkdir -p $HOME/projects

echo "Setup complete! Remember to 'source ~/.bashrc' to load the new environment variables"
```

## What Am I Looking At In Terms Of Profiling
1. Comparing the performance of chained hash table vs open addressing hash table under different workloads - look at criterion benchmarks for this
    * also try changing the hash implementation to see the impact.
2. Run a profile for both chained hashing vs open addressing using `perf`, `flamegraph` etc. to collect profiling data along with cache statistics.
3. Comparing the impact of different concurrency patterns on each hash table individually
    * coarse lock (single lock for the whole table)
    * granular locks (one lock per index)
    * striped locks (one lock per x indexes)
    It doesn't make sense to compare across implementations since I'm not really sure what that tells me.


## CPU & Cache Performance With Strings

### Strings

#### Cache Stats

```shell
  Performance counter stats for './target/release/hashmap -w load_factor -i chaining':

       762,544,185      cache-references                                                        (66.66%)
       597,421,905      cache-misses                     #   78.35% of all cache refs           (66.66%)
     8,145,732,959      L1-dcache-loads                                                         (66.67%)
       279,075,767      L1-dcache-load-misses            #    3.43% of all L1-dcache accesses   (66.68%)
       144,218,848      LLC-loads                                                               (66.67%)
       106,241,540      LLC-load-misses                  #   73.67% of all LL-cache accesses    (66.66%)

      14.427876578 seconds time elapsed

      13.234905000 seconds user
       1.191811000 seconds sys
  
  Performance counter stats for './target/release/hashmap -w load_factor -i open_addressing':

       654,770,201      cache-references                                                        (66.67%)
       509,578,087      cache-misses                     #   77.83% of all cache refs           (66.68%)
     5,511,667,961      L1-dcache-loads                                                         (66.67%)
       272,838,534      L1-dcache-load-misses            #    4.95% of all L1-dcache accesses   (66.67%)
        91,460,491      LLC-loads                                                               (66.66%)
        59,470,563      LLC-load-misses                  #   65.02% of all LL-cache accesses    (66.65%)

       9.068350211 seconds time elapsed

       7.816879000 seconds user
       1.250820000 seconds sys
```

#### CPU Stats

```shell
  Performance counter stats for './target/release/hashmap -w load_factor -i chaining':

                61      context-switches                 #    4.226 /sec
                 0      cpu-migrations                   #    0.000 /sec
    33,334,941,421      cycles                           #    2.309 GHz
    34,737,785,689      instructions                     #    1.04  insn per cycle
     6,377,780,492      branches                         #  441.853 M/sec
        37,015,167      branch-misses                    #    0.58% of all branches
         14,434.16 msec cpu-clock                        #    1.000 CPUs utilized
         14,434.20 msec task-clock                       #    1.000 CPUs utilized

      14.436026853 seconds time elapsed

      13.282750000 seconds user
       1.151891000 seconds sys

  Performance counter stats for './target/release/hashmap -w load_factor -i open_addressing':

                24      context-switches                 #    2.635 /sec
                 1      cpu-migrations                   #    0.110 /sec
    24,113,035,970      cycles                           #    2.647 GHz
    26,051,667,680      instructions                     #    1.08  insn per cycle
     4,472,654,897      branches                         #  491.004 M/sec
        30,880,291      branch-misses                    #    0.69% of all branches
          9,109.20 msec cpu-clock                        #    1.000 CPUs utilized
          9,109.21 msec task-clock                       #    1.000 CPUs utilized

       9.110056061 seconds time elapsed

       7.864538000 seconds user
       1.244926000 seconds sys
```

### u64

#### Cache Stats
```shell
Performance counter stats for './target/release/hashmap -w load_factor -i chaining':

       372,807,778      cache-references                                                        (66.65%)
       252,436,979      cache-misses                     #   67.71% of all cache refs           (66.65%)
     2,380,114,906      L1-dcache-loads                                                         (66.67%)
       140,770,087      L1-dcache-load-misses            #    5.91% of all L1-dcache accesses   (66.69%)
        72,878,214      LLC-loads                                                               (66.68%)
        41,523,145      LLC-load-misses                  #   56.98% of all LL-cache accesses    (66.66%)

       6.273809754 seconds time elapsed

       5.943481000 seconds user
       0.330026000 seconds sys

 Performance counter stats for './target/release/hashmap -w load_factor -i open_addressing':

       217,178,237      cache-references                                                        (66.63%)
       167,946,747      cache-misses                     #   77.33% of all cache refs           (66.64%)
       880,710,536      L1-dcache-loads                                                         (66.68%)
        81,384,415      L1-dcache-load-misses            #    9.24% of all L1-dcache accesses   (66.71%)
        23,916,052      LLC-loads                                                               (66.70%)
        14,262,560      LLC-load-misses                  #   59.64% of all LL-cache accesses    (66.65%)

       2.421592557 seconds time elapsed

       2.128437000 seconds user
       0.293060000 seconds sys

 Performance counter stats for './target/release/hashmap -w load_factor -i open_addressing_compact':

       171,399,933      cache-references                                                        (66.61%)
        86,106,253      cache-misses                     #   50.24% of all cache refs           (66.61%)
       955,677,380      L1-dcache-loads                                                         (66.66%)
        64,703,719      L1-dcache-load-misses            #    6.77% of all L1-dcache accesses   (66.73%)
        24,137,327      LLC-loads                                                               (66.73%)
         6,734,891      LLC-load-misses                  #   27.90% of all LL-cache accesses    (66.66%)

       1.617498666 seconds time elapsed

       1.411503000 seconds user
       0.205927000 seconds sys
```

#### CPU Stats
```shell
 Performance counter stats for './target/release/hashmap -w load_factor -i chaining':

                28      context-switches                 #    4.475 /sec
                 0      cpu-migrations                   #    0.000 /sec
    14,779,162,822      cycles                           #    2.362 GHz
    11,086,265,659      instructions                     #    0.75  insn per cycle
     1,825,037,986      branches                         #  291.655 M/sec
        32,027,771      branch-misses                    #    1.75% of all branches
          6,257.52 msec cpu-clock                        #    1.000 CPUs utilized
          6,257.54 msec task-clock                       #    1.000 CPUs utilized

       6.258278276 seconds time elapsed

       5.945761000 seconds user
       0.311987000 seconds sys

 Performance counter stats for './target/release/hashmap -w load_factor -i open_addressing':

                 6      context-switches                 #    2.495 /sec
                 0      cpu-migrations                   #    0.000 /sec
     6,878,076,102      cycles                           #    2.861 GHz
     5,917,571,925      instructions                     #    0.86  insn per cycle
       706,244,806      branches                         #  293.730 M/sec
        18,717,025      branch-misses                    #    2.65% of all branches
          2,404.40 msec cpu-clock                        #    1.000 CPUs utilized
          2,404.40 msec task-clock                       #    1.000 CPUs utilized

       2.404832297 seconds time elapsed

       2.106739000 seconds user
       0.297963000 seconds sys

 Performance counter stats for './target/release/hashmap -w load_factor -i open_addressing_compact':

                 4      context-switches                 #    2.503 /sec
                 0      cpu-migrations                   #    0.000 /sec
     5,303,061,825      cycles                           #    3.318 GHz
     6,293,209,416      instructions                     #    1.19  insn per cycle
       747,587,244      branches                         #  467.736 M/sec
        18,770,612      branch-misses                    #    2.51% of all branches
          1,598.31 msec cpu-clock                        #    1.000 CPUs utilized
          1,598.31 msec task-clock                       #    1.000 CPUs utilized

       1.598749446 seconds time elapsed

       1.415667000 seconds user
       0.182957000 seconds sys
```

## Size Of Data Structures

### With Strings

1. Chaining
    size of (K, V) -> 48
    size of linked list node -> 8

2. Open Addressing
    size of (K, V) -> 48
    size of entry -> 48

3. Open Addressing Compact
    size of (K, V) -> 48

### With u64

1. Chaining
    size of (K, V) -> 16
    size of linked list node -> 8

2. Open Addressing
    size of (K, V) -> 16
    size of entry -> 24

3. Open Addressing Compact
    size of (K, V) -> 16

## Tweet

1. 
I've been trying to understand profiling tools better and decided to build a cache conscious hash map as a way to get more familiar with the tooling. 

I wrote a small blog post which I'm going to cover in a 🧵
[insert image of cache.svg]

2. 
Here's the link to the blog post: [insert link to blog post once it's published]

3. 
My only real recommendation from the post is to not bother profiling on OSX! It's a locked down nightmare of an OS

I've covered all the steps required to get Linux up and running for profiling here: https://github.com/redixhumayun/learnings/issues/9

Apple requires jumping through too many hoops

4. 
Building a hash map is easy enough - a few public methods and a couple of internal methods to track when to resize the hash map

[insert image of external API]

5.
You can handle collisions via chaining where each entry becomes a linked list. This is simple but tends to lead to fragmented memory since each node is allocated separately.

Fragmented memory should theoretically lead to poorer cache performance.

[insert image of chained hash map]

6.
You can handle collisions via open addressing (linear probing). This is where you just search through every subsequent entry until you find one where you can insert the key-value pair.

This should be more cache friendly since the entire memory is contiguously allocated

[insert image of open addressing hash map]

7.
Profiling, however, shows very different results. It shows chaining and open addressing displaying roughly the same levels of cache performance.

Open addressing ends up performing better but that's only due to fewer overall instructions

[insert image of chaining vs open addressing cache stats vs CPU stats]

8.
The problem, as it turns out, is the memory layout of the structs used for chaining and open addressing

With chaining, we only have pointers which are 8 bytes each so that's 8 per cache line in a 64 byte cache line

With open addressing, it's 24 bytes each so that's 2 per cache line with padding

9.
We can do a more compact open addressing scheme which will fit 4 entries per cache line

Also, using u64 instead of Strings leads to better performance since Strings are always heap allocated

[insert image of compact scheme along with code showing the struct]

10.
Doing all of the above requires you to know some information about your hardware, like the size of your cache and the size of a cache line.

[insert screenshot of text from blog post showing the relevant section]

11.
Now, we see much better performance when comparing cache stats for the 3 implementations.

It isn't enough to have contiguous memory access, you also need to be aware of the cache hardware and how many entries can fit per cache line.

[insert image of profiling across 3 schemes]

12.
If you found this interesting, give the full article a read which goes into much more depth and contains links to more interesting tidbits around Rust compiler optimizations

[insert link to full article here]