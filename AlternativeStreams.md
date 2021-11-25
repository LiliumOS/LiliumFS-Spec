# Alternative Streams

1. Every PhantomFS object contains 1 or more Streams, which defines data and associated metadata of the Object. Each stream has a name, called an id, a set of flags, and associated data.

2. Recommended Practice: Operating Systems should make it possible to directly read (and, in some cases, write) the contents of an alternative stream. If a stream with id `Stream` is attached to an object with path `/foo/bar/baz`, then it should be possible to access that stream with the path `/foo/bar/$$baz$Stream`. If multiple streams with the same id exists, the first should be named `/foo/bar/$$baz$Stream$0`, the second `/foo/bar/$$baz$Stream$1`, etc. 

## Alternative Stream Listing

1. Each object shall have a Stream with id `Streams`. This stream contains the list of streams available on the object. The size of the stream shall be a whole number multiple of 128 bytes, divided into individual elements of 128 bytes, which are interpreted according to the following Rust structure

```rust
#[repr(C,align(128))]
pub struct StreamListing{
    name: [u8;32],
    name_ref: Option<NonZeroU64>,
    flags: u64,
    content_ref: u128,
    size: u64, 
    reserved: [u64; 3],
    inline_data: [u8;32]
}
```

2. The fields of the above structure shall be interpreted as follows:
    - name shall contain the id of the stream encoded in UTF-8, with all remaining bytes set to 0 if any, if the length (in bytes) of that name is at most 32 characters. Otherwise, it shall contain `0` bytes only.
    - name_ref shall be `None` if `name` contains a name. Otherwise, it shall be an index into the `Strings` stream of the current primary object, which is a Null Terminated UTF-8 string that is the id of the stream
    - flags shall be interpreted as a bitfield of the following bits:
         - 0x0000000000000001 (required): Correctly interpreting the stream is mandatory for accessing the file. Access to the current primary object MUST be denied if the implementation does not recognize the id of this stream
         - 0x0000000000000002 (write_required): Correctly intepreting the stream is mandatory for writing to this file. Write Access to the current primary object MUST be denied if the implementation does not recognize the id of this stream.
         - 0x0000000000000004 (enumerate_required): Correctly interpreting the stream is mandatory for enumerating the contents of this directory. Search access to the current primary object MUST be denied if the implementation does not recognize the id of this stream.
         - 0x0000000000000008 (preserve): This stream MUST be preserved even if the implementation cannot correctly interpret the stream. An implemenation MUST NOT modify the contents of the stream, or remove the stream when the file is accessed if it does not recognize the id of this stream. This bit is mutually exclusive with required and MUST NOT be set together. If the required bit is set, this bit is ignored. 
         - Exception: a stream for which the preserve bit is set may be removed at explicit request of the user if the Security Descriptor permits the action
         - 0x00000000000000*i*0 (indirection): The indirection level for the data within the stream, `0` means the data is contained within the `inline_data`. `1` means that `inline_data` is empty and `content_ref` refers directly to the data the stream contains, `2` means that `content_ref` refers to an array of pointers to the data the stream contains, etc.
         - 0xFFF0000000000000 (impl_use): Mask for bits which are available for implementation use. An implementation MUST NOT modify or interpret these bits unless it recognizes the `id` of the Stream. 
         - All other bits are reserved and access to the pimrary object MUST be denied if any such bit is set.
    - content_ref shall be an indecies of sector on the filesystem which contains the data of the stream if it is not included inline (indirection=`0`). If indirection>1, then the array of pointers refered to by `content_ref` and those pointers up to the indirection level shall be of discrete elements of type `u128` which themselves are indecies of sectors on the filesystem. 
    - size shall be the size, in bytes, of the data contained by the stream, not rounded to the nearest sector size
    - reserved shall be an array of all `0`s. Any other value in these fields MUST be ignored and MUST NOT be written.
    - inline_data contains the data of the stream if it is inline (indirection=0). Otherwise it is ignored and may contain any bytes.
3. A primary object may contain at most one stream with the id `Streams` and this stream MUST be the first element or access to the current primary object MUST be defined. The required bit MUST be set for the stream with id `Streams` and all impl_use bits MUST be clear. 
4. Recommened Practice: If a filesystem contains a file for which all access is denied by this section, the filesystem SHOULD fail to load if this can be detected. If a filesystem contains a file for which write access is denied by this section, the filesystem SHOULD fail to load in read-write configuration if this can be detected. 

## Long Strings Array

1. Each object may have a stream with the id `Strings`. A object may have at most one stream with the id `Strings`. The required bit of this stream MUST be set and all impl_use bits MUST be clear. 
2. The `Strings` stream contains an array of Null Terminated UTF-8 strings. The first byte of the stream MUST be `0` (an empty null-terminated string). The index `0` refers to this byte and acts as a sentinel value.


## Implementation Specific (Extended) Streams

1. An implementation may define extended streams that have an id that does not contain the `$` symbol and is not a standard stream name. The Phantom Filesystem reserves all stream names that consist of soley ASCII Letters and underscore characters for future use. Additionally, special cases of common unique id forms are reserved, as specified below.
2. An implementation that defines an extended stream may assign meaning to the impl_use bits of the `flags` field of the stream.
3. Recommended Practice: An implementation SHOULD ensure that extended streams are unique from other stream names, by assigning ids in forms that are likely to be unique, such as Universally Unique Identifiers (Optionally enclosed in braces), Uniform Resource Names with a unique prefix, Object Identifiers, or some other form that is likely to be unique.
4. The following id forms are reserved by this specification:
    - The Nil UUID `00000000-0000-0000-0000-000000000000`, optionally enclosed in braces
    - URNs that start with `phantomfs:`
    - OIDs that start with the prefix 1.3.6.1.4.1.54569.5.1

