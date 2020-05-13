# FerrisGC
Complete reimplementation of https://github.com/Manishearth/rust-gc from scratch providing just compatible interface

One of the main difference is a thread safe Garbage Collector implementation as well as Thread Local Garbage Collector 

Here is the simple example of using FerrisGC:
```rust
use ferris_gc::{Gc, Trace, Finalize, GcCell, ferris_gc_main, ApplicationCleanup};
use ferris_gc::sync::Gc as GlobalGc;
use ferris_gc::sync::GcCell as GlobalGcCell;
use core::time;
use std::thread;
use std::time::Duration;
use std::path::Path;
use std::fs::File;
use std::error::Error;
use std::io::Write;

#[derive(Trace)]
struct MyStruct {
    jh: u32,
}

impl Drop for MyStruct {
    fn drop(&mut self) {
        println!("MyStruct in drop !!");
    }
}

impl Finalize for MyStruct {
    fn finalize(&self) {
        println!("MyStruct in finalize !!");
        let mut file = File::create("foo.txt");
        match file {
            Ok(mut f) => { f.write_all(b"Hello, world!") },
            Err(e) => { Err(e) },
        };
    }
}

struct MyStruct3 {
    jh: u32,
}

#[derive(Trace, Finalize)]
struct MyStruct39(#[unsafe_ignore_trace] MyStruct3, u16);

#[derive(Trace)]
struct MyStruct2 {
    jh: Gc<u32>,
}

impl Drop for MyStruct2 {
    fn drop(&mut self) {
        println!("MyStruct in drop !!");
    }
}

impl Finalize for MyStruct2 {
    fn finalize(&self) {
        let mut file = File::create("foo2.txt");
        match file {
            Ok(mut f) => { f.write_all(b"Hello, world!") },
            Err(e) => { Err(e) },
        };
    }
}

#[ferris_gc_main]
fn main() {
    {
        let gc = Gc::new(2);
        let gc1 = Gc::new(MyStruct { jh: 3 });
        let gc2 = GcCell::new(MyStruct { jh: 3 });
        gc2.borrow_mut().jh = 2;
        let gc3 = Gc::new(MyStruct2 { jh: gc.clone() });
    }
    {
        let gc2 = GlobalGc::new(MyStruct { jh: 3 });
    }
    let gc3 = Gc::new(MyStruct { jh: 3 });
    let ten_secs = time::Duration::from_secs(20);
    thread::sleep(ten_secs);
    println!("Hello, GC World !!");
}
```

Lets try to understand step-by-step what is what here:
1) #[derive(Trace, Finalize)]
   Macro that implement Trace and Finalize for the particular object.
   It is needed because `Gc::new` and `GcCell::new` waiting object that implement trait `Trace` and using this trait GarbageCollector is able to collect unreachable objects ...
   Trait `Finalizer` will be called when object is disposed. Also you can implement custom trait `Finalizer` for your particular needs

2) #[ferris_gc_main]
   Macro for main function in your program is needed for disposing all objects after Application finished.
   It is usefull because otherwise object `gc3` will not be disposed by GarbageCollector and finalize will not be called ...

To add dependencies you should add:
```toml
[dependencies]
ferris-gc = { version = "0.1.2", features = ["proc-macro"] }
```
