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
    reserved33: [u8; 5],
    ty: u16,
    flags: u32,
    reserved44: [u8; 20],
}
```

3. The interpreation of the fields shall be as follows:
    - strong_ref shall be the number of hard links to the object (when the last strong reference is removed, all streams can be deallocated)
    - weak_ref shall be the total number of hard and weak links to the object (when the last weak reference is removed, the object can be fully deallocated)
    - streams_size is the size of the `Streams` stream, if the object's strong_count is > 0
    - streams_ref is index of the sector containing the `Streams` stream if the object's strong_count > 0. If streams_indirections > 1, then this refers to the indirection table for the appropriate level of indirection.
    - streams_indirection is the indirection level of the `Streams` stream. Always greater than `0` if the object's strong_count > 0 
    - reserved33 and reserved44 are all set to 0, and other values should be ignored
    - `ty` is the type of the object, and is interpreted as follows:
        - `0` (Regular File): The object primarily contains data, to be interpreted by programs opening the file. Regular files should have a "FileData" stream that contains these bytes
        - `1` (Directory): The object is primarily a directory that contains other files. Directories should have a "DirectoryContent" stream that contains the files
        - `2` (Symlink): The Object Primarily refers to the logical path of another object. In most cases, Symlinks are transparently replacable with the referent path. Symlinks should have a "SymlinkTarget" stream that contains the logical path as a UTF-8 stirng
        - `3` (Unix FIFO): The object is primarily a Named Pipe/FIFO object. 
        - `4` (Unix Socket): The object is primarily a Unix Socket. 
        - `5` (Block Device): The object is primarily a Block Device. Block Device Files should have a "DeviceId" stream or a "LegacyDeviceNumber" stream.
        - `6` (Character Device): The object is primarily a Character Device. Character Device Files should have a "DeviceId" stream or a "LegacyDeviceNumber" stream.

