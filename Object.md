# Objects 

1. Every file and directory in the phantomfs is uniformly described as objects. Objects describe basic information about how to access the information (if any) stored with the file.

2. Each object is described by the following 64 byte structure
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
    - weak_ref shall be the total number of hard and weak links to the object (when the last weak reference is removed, the object can be fully deallocated). If this value is `0`, the content of all other fields are undefined.
    - streams_size is the size of the `Streams` stream, if the object's strong_count is > 0
    - streams_ref is index of the sector containing the `Streams` stream if the object's strong_count > 0. If streams_indirections > 1, then this refers to the indirection table for the appropriate level of indirection.
    - streams_indirection is the indirection level of the `Streams` stream. Always greater than `0` if the object's strong_count > 0 
    - reserved33 and reserved44 are all set to 0, and other values should be ignored
    - `ty` is the type of the object, and is interpreted as follows:
        - `0` (Regular File): The object primarily contains data, to be interpreted by programs opening the file. Regular files should have a "FileData" stream that contains these bytes
        - `1` (Directory): The object is primarily a directory that contains other files. Directories should have a "DirectoryContent" stream that contains the files
        - `2` (Symlink): The Object Primarily refers to the logical path of another object. In most cases, Symlinks are transparently replacable with the referent path. Symlinks should have a "SymlinkTarget" stream that contains the logical path as a UTF-8 stirng
        - `3` (POSIX FIFO): The object is primarily a Named Pipe/FIFO object. 
        - `4` (Unix Socket): The object is primarily a Unix Socket. 
        - `5` (Block Device): The object is primarily a Block Device. Block Device Files should have a "DeviceId" stream or a "LegacyDeviceNumber" stream.
        - `6` (Character Device): The object is primarily a Character Device. Character Device Files should have a "DeviceId" stream or a "LegacyDeviceNumber" stream.
        - 65535 (Custom Type): The object has implementation-specific or custom semantics. Custom Type Objects should have a "CustomObjectInfo" stream.
        - other values are reserved and implementations MUST not allow access to objects with invalid types. 

4. Objects appear in an table within the filesystem, accessible from the filesystem information block. At most 18446744073709551615 total objects may exist within a single filesystem.

## Regular Files

1. Objects with a type of Regular File is a Regular, data carrying file. Opening the file by path accesses the content of the "FileData" stream if it exists.
2. An object may have a stream with an id of "FileData" and may have at most one such stream. Such a stream shall have the required bit set if it appears on an Regular File object, otherwise it may be set at the option of the implementation. No impl_bits may be set on the stream. The content of the FileData stream is exactly the content of the file as read and written, without any holes.

## Directories

1. Objects with a type of Directory is a directory containing other objects. Opening the file by path accesses the content of the "DirectoryContent" stream if it exists. Additionally, it is valid to refer to objects contained within the path using the directory separator on the platform (For example `/` on POSIX platforms), if the DirectoryContent stream exists.
2. An object may have a stream with an id of "DirectoryContent" and may have at most one such stream. Such a stream shall have the required bit set if it appears on a Directory object, otherwise it may be set at the option of the implementaiton. No impl_bits may be set on the stream. 
3. The content of the DirectoryContent stream is an array of the following structure:
```rust
#[repr(C,align(64))]
pub struct DirectoryElement{
    objidx: Option<NonZeroU64>,
    name_index: Option<NonZeroU64>,
    flags: u64,
    name: [u8;40]
}
```
4. The interpretation of the fields for DirectoryElement is as follows:
    - objidx is either None, or the index into the object table of the filesystem that refers to the allocated object
    - name_index shall either be None or an index in the "Strings" stream of the object, which is a NUll-Terminated UTF-8 String that is the name of the object within the directory. 
    - flags is a bitfield of the following:
        - 0x0000000000000001 (weak): The reference to the object within this directory is weak and does not keep hold of the content of the file. Removing this reference only decrements the weak count
        - 0x0000000000000002 (hidden): The file is hidden within the directory. This bit is a hint that indicates that UIs should not typically display this file to users unless enabled by user request (such as the `-a` flag for the POSIX ls command, or a "Show Hidden Files" option in a GUI-based file explorer).
        - Other flags are reserved and MUST NOT be set.
    - name contains the UTF-8 encoded name of the object within the directory if the length in bytes of that name is up to 40, with all trailing bytes (if any) set to zero. Otherwise, each byte in name shall be set to zero and the name is given by the name_index field.
5. When refering to objects within a path to an object, the contents of the object shall be resolved via the DirectoryContent stream of the file unless the file type is "Custom Type", which resolves objects via the default stream, or "Symlink" which resolves the contents of the object via logical path resolution to the target of the symbolic link


## Symbolic Links

1. Objects with a type of Symbolic Link are symbolic (logical) references to other files on the system. Opening this file follows the symbolic link to the path given by the "SymlinkTarget" stream, if any, unless the file is opened using an implementation-defined mechanism that does not follow the symbolic link. The details of such a mechanism, if any, are not specified.
2. An object may have a stream with an id of "SymlinkTarget" and may have at most one such stream. Such a stream shall have the required bit set if it appears on a Symbolic Link Object, otherwise it may be set at the option of the implementation. No impl_bits may be set on the stream.
3. The content of the SymlinkTarget stream is precisely the UTF-8 String that contains the path the symbolic link logically refers to, not including a NUL Terminator byte.
4. The result of opening the `SymlinkTarget` stream shall be the same as opening the path given by the "SymlinkTarget" stream, if any, unless the file is opened using an implementation-defined mechanism that does not follow the symbolic link
5. The result of following a Symbolic link that denies access to the `SymlinkTarget` stream is unspecified. 

## Nonstoring File Types

1. Two types are defined for compatibility with POSIX filesystem apis: FIFO and Unix Socket. No default stream is defined for these types and opening files of these types have the same handling behaviour as defined by POSIX.

## Device Files

1. Objects with the "Character Device" or "Block Device" type are device files. Opening such a file refers to an implementation-dependant byte stream identified by either the DeviceId stream or, if absent, the LegacyDeviceNumber stream.
2. The Character Device type is intended to allow access to devices which transfer data byte-by-byte (such as a serial connection). The Block Device typeis intended to allow access to devices which transfer data in large chunks (such as a disk drive).
3. The DeviceId stream contains the following 32 byte structure:
```rust
#[repr(C,align(16))]
pub struct DeviceId{
    devid_lo: u64,
    devid_hi: u64,
}
```
4. The devid_lo and devid_hi fields are the low and high 64 bytes of a 128-byte Universally Unique Identifier, devid.
5. The devid UUID has operating system and runtime-specific meaning, except the following devids are defined for the following purposes or should issue an error:
    - 0b503316-be4c-565d-8bd3-abb7d880915e (null): An empty stream that accepts and discards any writes.
    - 87ebc524-bcab-5e57-81d7-857abc57273b (full): A full, continuous stream of zero bytes that indicates no available space on writes.
    - 152725b2-bc57-56c4-81c4-a571afad9ef7 (zero): A continous stream of zero bytes that accepts and discards any writes.
    - 37b86a41-b1f2-574f-828e-6976586d8e68 (random): A stream of cryptographically strong, non-deterministic, random bytes. Bytes written to this stream affect it in an unspecified manner.
    - b36ef7fb-a502-5150-a75c-8334df7a5569 (urandom): Same as 37b86a41-b1f2-574f-828e-6976586d8e68, except that the bytes may instead be taken from a cryptographically strong pseudorandom number generator, rather than a non-determinstic random number source.
6. Recommended Practice: When a DeviceId is being used to refer to a partition on a disk formatted with a GUID Partition Table, the DeviceId should be set to the GUID of the Partition. 
7. The LegacyDeviceNumber stream contains the following structure:
```rust
#[repr(C,align(8))]
pub struct LegacyDeviceNumber{
    major: u32,
    minor: u32
}
```
8. The major and minor fields are the major and minor device numbers respectively. The resultant value has operating system and runtime-specific meaning.
