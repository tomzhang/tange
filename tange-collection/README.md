Tange-Collection
---
Tange-Collection is a medium-level dataflow library for high speed data processing.

What is it?
---
Tange-Collection provides dataflow operatores for quickly executing data processing tasks.  It uses task-based parallelization for construction of complex computation graphs, scalable to hundreds of millions of independent stages.

It was created to solve the same sort of processing tasks as Dask and Spark, with a higher
emphasis on batch processing rather than analytics.

API
---

* [Overall](https://docs.rs/tange-collection/0.1.0/tange_collection/)
* [MemoryCollection](https://docs.rs/tange-collection/0.1.0/tange_collection/collection/memory/struct.MemoryCollection.html)
* [DiskCollection](https://docs.rs/tange-collection/0.1.0/tange_collection/collection/disk/struct.DiskCollection.html)

Example - Word Count
---

```rust
extern crate tange;
extern crate tange_collection;

use tange::scheduler::GreedyScheduler;
use tange_collection::utils::read_text;

use std::env::args;

fn main() {
    let path = args().nth(1).unwrap();
    let col = read_text(&path, 4_000_000)
        .expect("File missing");

    let graph = col 
        .map(|line| line.split_whitespace().fold(0usize, |a,_x| a + 1)) 
        .fold_by(|_count| 1,
                 || 0usize,
                 |acc, c| { *acc += c },
                 |acc1, acc2| { *acc1 += acc2 },
                 1);
    
    if let Some(counts) = graph.run(&GreedyScheduler::new()) {
        println!("Counts: {:?}", counts);
    }   
}
```
Example - IDF count
---
```rust
extern crate tange;
extern crate tange_collection;

use tange::scheduler::GreedyScheduler;
use tange_collection::utils::read_text;

use std::env::args;
use std::collections::HashSet;

fn main() {
    env_logger::init();
     
    let path = args().nth(1).unwrap();
    let col = read_text(&path, 64_000_000)
        .expect("File missing");

    let total_lines = col.count();
    let word_freq = col
        .emit_to_disk("/tmp".into(), |line, emitter| {
            let unique: HashSet<_> = line.split_whitespace().map(|p| p.to_lowercase()).collect();
            for word in unique {
                emitter(word);
            }
        })
        .frequencies(16);

    // Cross product
    let idfs = total_lines.join_on(
            &word_freq.to_memory(),
            |_c| 1,
            |_wc| 1,
            |total, (word, count)| {
                (word.clone(), (1f64 + (*total as f64 / *count as f64)).ln())
            },  
            1   
        )
        .map(|(_k, x)| x.clone())
        .sort_by(|(word, _count)| word.clone());

    if let Some(word_idf) = idfs.run(&GreedyScheduler::new()) {
        for (w, idf) in word_idf {
            println!("{}: {}", w, idf);
        }
    }
}
```


