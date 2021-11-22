# Objects 

1. Every file and directory in the phantomfs is uniformly described as Objects. Objects describe basic information about how to access the information (if any) stored with the file.

2. Each Object is described by the following 64 byte structure
```rust
#[repr(C,align(64))]
pub struct Object{
    strong_ref: u32,
    weak_ref: u32, 
    streams_size: u64,
    streams_ref: u128,
    streams_indirection: u8,
    reserved: [u8; 5],
    ty: u16,

}


