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

In the above logs, we can see two calls to `mmap` and two calls to `munmap`. It seems to roughly map to this code

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