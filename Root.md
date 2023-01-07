# Filesystem Root

## Root Descriptor Table

At byte offset 1024 in the volume (In the case of a disk with 512 byte sectors, this is at sector offset 2, not sector offset 1), a root descriptor table is placed, that describes properties of the filesystem.

Every PhantomFS filesystem contains exactly one root descriptor table, at this specified offset.


```rust
#[repr(C,align(128))]
pub struct RootDescriptor{
    pub magic: [u8;4],
    pub version_major: u16,
    pub version_minor: u16,
    pub required_features: u32,
    pub optional_features: u32,
    pub volume_id_lo: u64,
    pub volume_id_hi: u64,
    pub root_object_id: Option<NonZeroU64>,
    pub objtab_size: u64,
    pub objtab_end: u128,
    pub alloc_tab_size: u64,
    pub alloc_tab_begin: u64,
    pub label_ref: Option<NonZeroU64>,
    pub label: [u8; 32],
    pub header_size: u32,
    pub crc: u32
}
```

`magic` contains the exact bytes `[F3 50 48 53]`. This is the magic number that identifies a PhantomFS Filesystem. 

`version_major` and `version_minor` contains the version field. For the major version feld, it is the defined major version minus one, and the minor version field is the defined minor version.
The current version of the spec is `1.0`, encoded with a `verson_major` field of `0`, and a `minor_version` field of `0`.

`required_features` and `optional_features` correspond to features not in the original spec that the filesystem uses. If a bit of `required_features` is not recognized by the implmentation, the implementation must not load the filesystem. If a bit of `optional_features` is not recognized, it must be ignored, but no unrecognized bits of either field should be modified by an implementation that rewrites the `RootDescriptor` (unless the implementation is creating a new filesystem on an existing volume). 

`volume_id_lo` and `volume_id_hi` together contain the 128-bit UUID of the volume. This is set by the implementation that creates the filesystem and should not be modified.
In the case of a filesystem stored in a partition of a GPT disk, it is recommended that this be the same as the partition id stored in the GPT. 
The fields are concatenated in memory order and read as a single little-endian 128-bit value, which contains the UUID. [Note: This is not the same layout as the GUIDs in a GPT.]

`root_object_id` is the object id of the root directory. If it is `None`, this file sytem shall not be loaded, but may be modified without creating a new filesystem. 
[Note: By convention, when creating a new filesystem, this should be set to `1`, denoting the first object on the filesystem - this is not a requirement, and implementations must accept root descriptors with any object id for the root object].

`objtab_size` is the size (in bytes) of the object table. This is a multiple of the size of the `Object` structure (32 bytes).
`objtab_end` is the sector number of the first sector following the end of the object table. The object table grows downwards from this point, through to `objtab_size`.

`alloc_tab_size` is the size of the allocation table, in bytes, which is a multiple of 32 bytes.
`alloc_tab_begin` is the sector number of the first sector used by the allocation table.

`label_ref` is the offset into the `Strings` stream of the root object that refers to the null terminated UTF-8 string that contains the disk label, or `0` if the disk label is at most 32 bytes in size.
`label` is the UTF-8 string that contains the disk label, if it is at most 32 bytes in size, padded with `0` bytes if it is less than 32 bytes. Otherwise, it contains all `0`s. [Note: If the string is exactly 32 bytes long, it will be inlined into this field, but will not have a nul terminator].

`header_size` is the number of bytes that constitutes the full root descriptor table. This is at least the size of the `RootDescriptor` type, but may be bigger when reading filesystems created by later versions. The entirety of the header must be preserved. When modifying the `RootDescriptor` of a filesystem (but not creating a new one), an implementation must not modify any bytes belonging to the header but not part of the `RootDescriptor` object. 
`crc` is the CRC32 (Polynomial `0x04C11DB7`) of the `RootDescrptor` object, not including any tail bytes that are part of `header_size` but not part of the `RootDescriptor`.


### Allocation Table

The allocation table stores an array of Volume Spans, representing allocations of the filesystem. 

```rust
#[repr(C,align(32))]
pub struct VolumeSpan{
    pub base_sector: u128,
    pub extent: u64,
    pub reserved: [u8;8],
}
```

The `base_sector` field is the first sector occupied by the span. If it is set to `0`, then the span is unused (and can be allocated).
`extent` is the number of bytes the span occupies. Note that if the `extent` occupies less than a multiple of 1024 bytes, the entire sector is still reserved to that span. Allocations can only start on a sector boundary. `reserved` shall be set to `0`

The `VolumeSpan` type is also used for indirections.



