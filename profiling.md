## Running With strace

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

To instrument for allocations -> `xcrun xctrace record --template "Allocations" --launch ./target/debug/hashmap -o ./profile_data`
To instrument for time profiling -> `xcrun xctrace record --template "Time Profiler" --launch ./target/debug/hashmap -o ./profile_data`