# Additional supported Streams

1. The following describes streams that are recommended for any implementation to support, but not required.

## Directory Organization

1. Two types of special organization structure are provided for the `DirectoryContent` stream, to allow more efficient lookup of files in the directory by their name. Support of either stream is OPTIONAL.

2. The `DirectoryContentHash` stream is used to perform hashtable lookup on the DirectoryContent stream. The id of the stream is `DirectoryContentHash` and the WRITE_REQUIRED bit shall be set. The PRESERVE, REQUIRED, and ENUMERATE_REQUIRED bits shall *not* be set and no impl_bits shal be set. The stream contains a 64 byte header, followed by an array of `u64`, which are indecies in the `DirectoryContent` stream. The string is hashed according to the header, and then taken modulo the number of entries (stream size-64)/8, and used to index this array. 
The resulting index is then the first index of a `DirectoryContent` entry with a matching name, which may then be iterated from until either the correct file is found or an empty (objidx=None) entry is found (in which case, the file is not present). The `DirectoryContentHash` stream requires putting an empty entry in the first entry of the directory (Which is index `0`). This allows empty buckets to use the entry `0` as a sentinel to avoid additional disk access. 
3. If a `DirectoryContentHash` stream exists and the implementation does not support it, the `DirectoryContent` stream may be modified only if the the `DirectoryContentHash` stream is removed. Additionally, if the directory content stream is malformed (Because the implementation does not recognize the algorithm or a flag is set that is not defined), the `DirectoryContent` stream may be modified only if the `DirectoryContentHash` stream is removed or replaced. If the stream is recognized and valid, the `DirectoryContent` stream may only be modified in compliance with the structure of the `DirectoryContentHash` stream, unless the stream is removed or replaced. 

3. The header of the `DirectoryContentHash` stream is given by the following structure:
```rust
#[repr(C,align(64))]
pub struct DirectoryContentHashHeader{
    pub alg: u64,
    pub flags: u64,
    pub seed: [u64;6],
}
```

4. The `alg` field is an offset into the `Strings` stream of the current object, which denotes the name of the hash algorithm. If the hash algorithm is not recongized, the stream is malformed. Algorithm names that consistent solely of Ascii letters and digits are reserved for this spec, other names are free for implementation use. The following predefined algrithm names should be recognized, but support of any particular algorithm is not required:
* `FNV1a64` - the FNV-1a hash with the standard 64-bit seed and prime, using a 64-bit accumulator. 
* `FNV1a64Seeded` - the FNV-1a hash with a standard 64-bit prime and using a 64-bit accumulator, but using the first entry of the `seed` array
* `FNV1a128` - The FNV-1a hash with the standard 128-bit seed and prime, using a 128-bit accumulator. The value is not truncated prior to being taken modulo the array length
* `FNV1a128Trunc` - The FNV-1a hash with the standard 128-bit seed and prime, using a 128-bit accumulator. The value is truncated to 64-bit immediately prior to being taken modulo the array length. 
* `FNV1a128Seeded` - The FNV-1a hash with the standard 128-bit prime, using a 128-bit accumulator, but using the first two entries of the `seed` array in little endian byte order. The value is not truncated prior to being taken modulo the array length
* `FNV1a128TruncSeeded` - The FNV-1a hash with the standard 128-bit prime, using a 128-bit accumulator, but using the first two entries of the `seed` array in little endian byte order. The value is truncated to 64-bit immediately prior to being taken modulo the array length. 
* `XXHash64` - The xxhash64 hash, using the first element of the seed array. 
* `SipHash4-2` - The SipHash algorithm, with 4 compression and 2 finalization rounds, and a 64-bit result, using the first 2 elements of the `seed` array.

5. It is not specified how the implemntation chooses which algorithm it generates the hashtable with. It is recommended that, where fiesible, the implementation give the user mounting the filesystem in a particular directory structure the option of which algorithm to use from among those it supports, both standard and implementation-specific.

6. The `flags` field is reserved for use as a flags field for future versions. NO flags are currently defined, so the field shall be set to `0` and if set to any other value, the structure is malformed.

7. The `seed` field is used to store initial parameters depending on the hash function in use. Up to 6, 64-bit values are available. Any element that is not used by the function is unused and may be set to any value. 

8. Normative References: [XXHash64](https://github.com/Cyan4973/xxHash/blob/dev/doc/xxhash_spec.md#xxh64-algorithm-description), [FNV-1a](http://www.isthe.com/chongo/tech/comp/fnv/), [SipHash](https://www.aumasson.jp/siphash/siphash.pdf)

## Partitioned Files/Devices

1. The `PartitionedDevice` stream can be used to view the content of a device or a raw file that contains a partition table as a directory of partitions, which may then be viewed as raw files or directories containing the filesystem on the partition, as the case may be.
2. There shall be at most one `PartitionedDevice` stream on an object, the REQUIRED, WRITE_REQUIRED, ENUMERATE_REQURIED, and PRESERVED bits shall not be set, and no `impl_bits` shall be set. The stream shall contain the following 32-byte structure:
```rust
#[repr(align(32))]
pub struct PartitionedDeviceInfo{
    pub target_stream: u64,
    pub flags: u64,
    pub reserved: [u64;2]
}
```

3. `target_stream` shall be index into the `Streams` array that is the stream tha contains the data being referred to. This specification defines the behaviour on `FileData` streams, `DeviceId` streams, and `LegacyDeviceNumber` streams. Implementations that define additional implementation-defined streams that provide raw binary data are encouraged to support using them with `PartitionedDevice`. Using the stream on any other standard stream is an error, and attempting to enumerate the `PartitionedDevice` stream or use it in path resolution shall fail. The flags field is reserved for flags, and must be clear. The `reserved` fields must be set to `0`. 

4. The PartionedDevice stream may be viewed as a directory, when enumerated, or used in path resolution, it may be viewed as containing a list of objects, with the names of the objects being the integer indecies into the partition table (MBR or GPT). Additionally, the implementation is recommended to view the same symbolic objects through the partition UUID in the case of a device with a GUID PArtition Table, and additionally the filesystem label if the filesystem of the partition is recognized and the user-readable label can be determined. 

5. The Objects shall act as though they have a an appropriate `Streams` stream, `Strings` stream, `SecurityDescriptor` stream, and either a `FileData` stream or a `DirectoryContent` stream, the choice of which is given below. Both a `FileData` stream and a `DirectoryContent` stream may appear at the option of the implementation. The type of the object is either a Regular File or a Directory, and the choice of which is given below. The `SecurityDescriptor` stream shall have an object owner row that is the same as the object the `PartionedDevice` stream is applied to, and a DEFAULT row applied to all streams, and the permission `"*"`, set to INHERIT. It is implementation-defined whether each of these symbolic objects may be assigned a valid id, and whether the id may be referenced by any other directory entry. 

6. If the partition contains a valid PhantomFS filesystem, then the object shall have type `Directory` and shall contain a `DirectoryContent` stream which enumerates over the objects in the root object. If it contains any other recognized kind of implemention-defined filesystem, it should expose the content of that filesystem in an implementation-defined manner, and setting the type to `Directory` with an appropriate `DirectoryContent` stream. It is implementation-defined whether each of the symbolic objects present on each filesystem may be assigned a valid id, and whether the id may be referenced by any other directory entry. Otherwise, it has type `RegularFile` and a `FileData` stream that contains the raw bytes of the partition.

7. For Partitions containing `PhantomFS` filesystems, the objects viewed have the same streams as are present in the filesystem, except that the root object may have the same SecurityDescriptor as the partition and it is implementation-defined whether the `InstallSecurityContext` stream on any object or other, implementation-defined, extended streams are effective or whether they may be viewed or modified. For other partitions, the streams viewed on each object are implementation-defined. 

[Note: As an example, an implementation may support the Linux ext2 filesystem, and expose a `LegacySecurityDescriptor` stream, rather than a `SecurityDescriptor` stream, correponding to the unix permissions of each object]

8. It is implementation-defined whether an object with a `PartionedDevice` stream, and no `DirectoryContent` stream may be used directly in path resolution and enumeration without an explicit stream reference. 

9. If a symbolic object that represents a partition has type `RegularFile`, it is implementation defined whether the partition may be viewed as a directory immediately after writing a recongized filesystem to the `FileData` stream of that object. If the file has type `Directory`, it is implementation-defined whether the `FileContent` stream is present and may be modified, and whether modifying the `FileContent` stream will reflect immediate changes on the objects viewed by the `DirectoryContent` stream.


## Additional Object Metadata

1. The `Metadata` stream stores additional data about the object, which provides miscellaneous information about the file. 
2. The stream type is `"Metadata"`. Whether or not the `REQUIRED`, `WRITE_REQUIRED`, or `ENUEMRATE_REQUIRED` bits are set is unspecified. If the `REQUIRED` bit is not set, the `PRESERVED` bits shall be set. 
3. The content of the `Metadata` stream is an array of the following elements, where the length of the array is the size of the stream divided by the size of the structure
```rust
#[repr(align(32))]
pub struct MetadataElement{
    type_ref: Option<NonZeroU64>,
    inline_type: [u8;16],
    data_tys: [u8;8],
    data: [u8;64]
}
```

4. `type_ref` is either `None` or the offset into the `Strings` stream, designating the name of the metadata type
5. `inline_type` is either set to `0` or is the name of the metadata type in UTF-8, if it is at most 20 bytes long, with remaining bytes set to `0`. 
6. `data_tys` stores information about the fields (in order) stored in `data`. Up to 8 fields total may be stored. Each type is one of:
    * `0` (no type): No more data. Remaining fields must be set to `0`
    * `1` (`u8`): A 1 byte integer, signed (2s complement) or unsigned
    * `2` (`u16`): A 2 byte integer, signed (2s complement) or unsigned
    * `3` (`UUID`): A uuid stored as 2 `u64`s
    * `4` (`u32`): A 4 byte integer, signed (2s complement) or unsigned
    * `5` (`Offset`): An Offset or Duration, stored as a i64 seconds since the UNIX epoch, followed by subsecond nanos
    * `8` (`u64`): An 8 byte integer, signed (2s complement) or unsigned
    * `32+n` (`String`): A String, stored as a 8 byte optional index into the string table, and an `n` byte inline string, `0<=n<=32`
    * All other types are reserved and any metadata that specifies a field of a reserved type must be ignored.
7. `data` is the data the metadata object contains, with layout given by `data_tys`. 
8. There shall be at most one `Metadata` stream per object.

## Stream Metadata

1. The `StreamMetadata` stream stores additional data about streams attached to an object, which provides miscellaneous information about those streams.
2. The stream type is `StreamMetadata`. Whether or not the `REQUIRED`, `WRITE_REQUIRED`, or `ENUEMRATE_REQUIRED` bits are set is unspecified. If the `REQUIRED` bit is not set, the `PRESERVED` bits shall be set. 
3. The content of the `StreamMetadata` is the following 32 byte header, followed by an array of the `MetadataElement` type defined above, where the length of the array is given by the remaining size of the stream, divided by the element size.
```rust
#[repr(align(64))]
pub struct StreamMetadataHeader{
    target_stream: u64,
    reserved: [u64;3]
}
```

4. `target_stream` is set to the index in the streams array designating the stream that the metadata stream applies to. [Note: This may be set to `0` and designate the `Streams` stream]
5. `reserved` shall be set to `0`.
6. There may be any number of `StreamMetadata` streams. 