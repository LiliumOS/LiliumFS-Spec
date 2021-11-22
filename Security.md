# Security 

This section describes the Security features of the Phantom Filesystem, including Security Descriptors, DACLs, and protected alternative streams.


Normative References: Alternative Streams

## Security Descriptors

1. A primary object may have a stream with the id "SecurityDescriptor". A primary object may have at most one such stream. Such a stream MUST have the required bit set, and all impl_bits shall be clear. 
2. The SecurityDescriptor stream shall contain an array of 64 byte elements, described by the the following Rust type:
```rust
#[repr(C,align(64))]
pub struct SecurityDescriptorRow{
    principal: u128,
    permission_name_ref: Option<NonZeroU64>,
    flags_and_mode: u64,
    permission_name: [u8; 32]
}
```
3. The fields of each element of the array shall be interpreted as follows:
    - principal shall be set to the Binary representation of the Universally Unique Identifier the creating system assigned to the principal that is identified by this row, all zeroes if the principal identified is the system, or all ones to indicate a default row. 
    - permission_name_ref shall either be None, or be an index into the `Strings` stream which is a null terminated UTF-8 Stirng that is the name of the permission item.
    - flags_and_mode shall be interpreted as follows:
        - 0x00000000000000mm (mode): One byte is reserved for the mode. The value `0` for mode indicates "PERMIT" (The permission is granted to the principal and it's subordinate principals), the value `1` indicates "DENY" (The permission is denied), the value `2` indicates "FORBID" (The permission is denied and cannot be restored by a different row), and the value `3` indicates "INHERIT" (inherit the permission from the parent directory being used to access the object). Other values of mode are reserved.
        - 0x0000000000000100 (required): Indicates that the permission is required for the security of the