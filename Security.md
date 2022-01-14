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
    stream_id: Option<NonZeroU64>,
    flags_and_mode: u64,
    permission_name_ref: Option<NonZeroU64>,
    permission_name: [u64;24]
}
```
3. The fields of each element of the array shall be interpreted as follows:
    - principal shall be set to the Binary representation of the Universally Unique Identifier the creating system assigned to the principal that is identified by this row, all zeroes if the principal identified is the system, or all ones to indicate a default row. 
    - stream_id: The index into the `Streams` stream which indicates the stream the DACL entry applies to, or None if it applies to the entire object 
        - [Note: `0`, which is the value that represents `None`, is the `Streams` entry of the Streams table, thus open to affect the whole Object.]
    - permission_name_ref shall either be None, or be an index into the `Strings` stream which is a null terminated UTF-8 String that is the name of the permission item.
    - permission_name shall contain the UTF-8 string that is the name of the permission item if it is at most 24 bytes, terminated by a null byte if there is less than 24 bytes, or otherwise 
    - flags_and_mode shall be interpreted as follows:
        - 0x00000000000000mm (mode): One byte is reserved for the mode. The value `0` for mode indicates "PERMIT" (The permission is granted to the principal and it's subordinate principals), the value `1` indicates "DENY" (The permission is denied), the value `2` indicates "FORBID" (The permission is denied and cannot be restored by a different row), and the value `3` indicates "INHERIT" (inherit the permission from the parent directory being used to access the object). Other values of mode are reserved.
        - 0x0000000000000100 (required): Indicates that the permission is required for the security of the filesystem, and an operating system should deny access to any file with a permission isn't doesn't recognize. The required bit may be ignored if the file is being accessed in a way that normally circumvents security checks.
        - 0xii00000000000000 (impl_bits): Additional bits for use with implementation-specific permissions.
        - All other flags are reserved and must not be set.
4. *Recommended Practice*: If the operating system does not support enhanced principals and permissions, then a principal representing a unix user should be given by the Version 3 UUID computed from the URI `2b6f4d63-7f84-53be-ab0f-9b4c1d7bf55a:Users/<uid>` where *uid* is the ascii decimal representation of the legacy unix user id, except that uid `0` should exactly use the UUID `00000000-0000-0000-0000-000000000000`, a principal representing a unix group should be given by the Version 3 UUID computed from the URI `2b6f4d63-7f84-53be-ab0f-9b4c1d7bf55a:Groups/<gid>`, where *gid* is the ascii decimal representation of the legacy unix group id. This practice is optional apply if the operating system supports PhantomOS Enhanced Principals and supports mapping between legacy user and group ids and PhantomOS Enhanced Principals.
5. [Note: If the SecurityDescriptor stream denies read access to either the SecurityDescriptor stream or the LegacySecurityDescriptor stream of a file to a particular thread, this shall not prevent either stream from being consulted to determine the access rights of a file (but does prevent direct read access to the stream, and thus may prevent discovery of the explicit permission rows in the stream)]


### Well-known Permissions

1. Several permission names are well-known and provided by this spec, and are provided here. These permissions must be supported by the implementation and must have the required bit set when present, and must not have any impl_bits set.
2. The "ObjectOwner" permission is a symbolic permission that indicates the owner of the object, if any. At most one ObjectOwner permission may appear in the SecurityDescriptor stream of the file, it must be applied to the entire object, must use mode 0 (PERMIT), and may not mention the DEFAULT principal (all 1 bits in the principal field). This entry need not be present.
    - [Note: The owner of an object has special permissions in that they can always read and write the SecurityDescriptor stream of the file, regardless of any other permission denying that access]
3. The "Read" permission indicates whether a particular principal can read the object or the indicated stream.
4. The "Write" permission indicates whether a particular principal can write to an object or the indicated stream.
5. The "Execute" permission indicates whether a particular principal can execute an object or the indicated stream.
6. The "AccessDirectory" permission indicates whether a particular principal can search the contents of the directory given by the object or the indicated stream. This permission only makes sense on objects with the `Directory` type, on the `DirectoryContent` stream of an object, or on some other stream that permits access to the directory. This permission is ignored when applied directly to an object that doesn't default to opening a directory from the default stream.
7. The "TakeOwnership" permissions indicates whether a particular principal can create or modify the `ObjectOwner` permission entry by setting `principal` to either the principal, or a group the principal belongs to. THis also allows modification of the `sd_uid;` and `sd_gid;` fields of the LegacySecurityDescriptor stream in the same manner. This permission entry only has effect when present on either the object, on the SecurityDescriptor stream, or on the LegacySecurityDescriptor stream.
    - [Note: This operation is valid regardless of whether the principal has write or even read access to the relevant stream. It may not be valid to perform the modification directly, in which case, the operating system should expose a separate filesystem  operation for this purpose.]
    - [Note: Regardless of which stream, if any, the permission applies to, both operations may be made available nevertheless, however, they may also be denied if they apply]
    - [Note: this permission is not implied by possessing Write access to the SecurityDescriptor stream]
8. The CreateObject permission indicates whether a particular principal can create objects within a directory or directory-like object. This permission only makes sense on objects with the `Directory` type, on the `DirectoryContent` stream of an object, or on some other stream that permits access to the directory. This permission is ignored when applied directly to an object that doesn't default to opening a directory from the default stream. This is implied by `Write` permission to the stream this is applied to, and `Write` permission to the object if the Default stream of that is the stream this permission applies to.
9. The RemoveObject permission indicates whether a particular principal can remove objects within a directory or directory-like object. This permission only makes sense on objects with the `Directory` type, on the `DirectoryContent` stream of an object, or on some other stream that permits access to the directory. This permission is ignored when applied directly to an object that doesn't default to opening a directory from the default stream. This is implied by `Write` permission to the stream this is applied to, and `Write` permission to the object if the Default stream of that object is the stream this permission applies to.
10. A special permission "*" indicates the wildcard permission. This row applies to all permissions available to the current system.
11. When querying a permission, if an entry is unavailable for the object, then DENY is implied. If an entry is unavailable for a particular stream, then PERMIT is implied if PERMIT is granted for the object except as follows:
    - Any permission applied to the SecurityDescriptor or LegacySecurityDescriptor, or other implementation-specific streams responsible for controling security access to the file defaults to DENY.
12. Permissions are queried from rows starting from index `0` and in increasing order. If one row contains a PERMIT that permits a particular principal, then a subsequent row contains a DENY or FORBID that permits a particular principal, then the latter row applies and the other row is ignored. Likewise, if one row contains a DENY, and a subsequent row contains a PERMIT, the latter row applies. This includes if the rows dow not overlap in the principal field. A FORBID entry overrides this rule. If a FORBID entry applies to a principal, then it controls, no subsequent PERMIT entry applies.
13. When applying permissions to access a stream, the permissions on the object are consulted first, then the permissions on the particular stream. Rows on the stream override rows on the object as above.
14. The owner principal of an object (given by the `ObjectOwner` permission) always has Read and Write access to the SecurityDescriptor and LegacySecurityDescriptor streams, and always has the TakeOwnership permission, regardless of any row present in the SecurityDescriptor (even a FORBID row)
15. A DEFAULT row Applies only if no other row applies to the context, regardless of order. a FORBID DEFAULT row does not prevent a specific PERMIT row from applying. 
    - [Note: For example, the following DACL will grant `Foo` permission to access the file, but Forbid `Bar`
    ```
    FORBID DEFAULT Read
    PERMIT Foo Read
    ObjectOwner Bar
    ```
    ]
16. A wildcard permission row applies even if another row mentions a specific permission by name, except that a wildcard permission row does not affect the ObjectOwner permission. 

## Legacy Security Descriptor

1. Each object may have a LegacySecurityDescriptor stream. The stream shall have the required bit set. Each object may have at most one LegacySecurityDescriptor stream.
2. The LegacySecurityDescriptor stream contains the following structure
```rust
#[repr(C,align(16))]
pub struct LegacySecurityDescriptor{
    sd_uid: u32,
    sd_gid: u32,
    sd_mode: u16,
    sd_reserved: [u8;6]
}
```
3. The LegacySecurityDescriptor provides a mechanism to represent legacy unix permissions which apply in the absence of a SecurityDescriptor.

4. `sd_uid` contains the id of the owning unix user. If the operating system supports only legacy unix users/groups, or supports mapping from legacy unix users/groups to enhanced phantomos principals, then if the SecurityDescriptor stream does not exist, or does not contain an ObjectOwner row, then the user identified by the `sd_uid` field is  considered the owner of the object. 
5. `sd_gid` contains the id of the owning unix group. If the operating system supports only legacy unix users/groups, or supports mapping from legacy unix users/groups to enhanced phantomos principals, then if the SecurityDescriptor stream does not exist, or does not contain an ObjectOwner row, then the user identified by the `sd_gid` field is  considered the owner of the object. 
    - [Note: Both the user identified by `sd_uid` and the group identified by `sd_gid` will be considered owners of the file]
    - [Note: Under normal unix legacy permissions, it is impossible for a process to execute in the security context of a group, thus this assignment is largely irrelevant.]
6. `sd_mode` contains the legacy unix permission field in the bottom 12 bits, with the upper 4 bits clear. If the object does not have a SecurityDescriptor stream, then this field is used, in combination with sd_uid and sd_gid, to determine the permissions (permissions are computed according to unix). Note that the suid and sgid fields are used even if the SecurityDescriptor stream exists
    [Note: The mapping from sd_mode maps the r bits to the corresponding Read permission, the w bits to the corresponding Write permission, and the x bits to the corresponding Execute (for non-Directory objects) or AccessDirectory (for Directory objects) permission. Thus, as an example, the permission field 755 on a file corresponds to the Security Descriptor:
    ```
    PERMIT FromUid(sd_uid) Read
    PERMIT FromUid(sd_uid) Write
    PERMIT FromUid(sd_uid) Execute
    PERMIT FromGid(sd_uid) Read
    PERMIT FromGid(sd_uid) Execute
    PERMIT DEFAULT Read
    PERMIT DEFAULT Execute
    ```
    ]
7. When executing a program with a LegacySecurityDescriptor stream, the SUID and SGID bits should modify the security context of the process created as a result. In particular, 
    - If the SUID bit is set, and the operating system supports either legacy unix users/groups, or supports mapping legacy unix users/groups to enhanced phantomos principals, then the process shall execute with an effect security context that indicates a primary principal of the user mentioned by the `sd_uid` field.
    - If the SGID bit is set, and the operating system supports either legacy unix users/groups, or supports mapping legacy unix users/groups to enhanced phantomos principals, then the process shall execute with an effect security context that indicates a secondary principal of the group mentioned by the `sd_gid` field. The security context shall have membership in the group.
    - If the SGID bit modifies the security descriptor of a process created by executing a program with a LegacySecurityDescriptor stream, then the Execute permission of the object shall be rechecked with the resulting security descriptor before the process begins execution 
    - [Note: this is intended to preserve "locked" binaries, with mode `g+s,g-x`]
8. When Creating an object within a directory that has a LegacySecurityDescriptor stream, the SUID and SGID bits modify the created object. In particular, if either SUID or SGID is set, then creating the object creates a LegacySecurityDescriptor with an unspecified `sd_mode`, but none of SUID, SGID, nor sticky shall be set. If the SUID bit is set, then `sd_uid` is set to the `sd_uid` field of the directory's LegacySecurityDescriptor. If the SGID bit is set, then `sd_gid` is set to the `sd_gid` field of the directory. Otherwise, the initialization of the respective fields is implementation-specific. An ObjectOwner row should be not be automatically created in the SecurityDescriptor stream of the new object if one is created, when either SUID or SGID is set.
9. The Sticky bit of `sd_mode` modifies the permission checks performed when removing a file from a Directory. If a LegacySecurityDescriptor stream is present on an object with a default stream of `DirectoryContent`, or some other implementation-specific stream that opens as a directory, then when checking the `RemoveObjects` permission of that directory to determine access rights to remove a file, then if the stick bit is set, and either a LegacySecurityDescriptor is present on the object being removed *or* the SecurityDescriptor, if present, contains an ObjectOwner row, then the primary principal must match one of the owners provided by the ObjectOwner permission or by the sd_uid/sd_gid fields or the operation should deny permissions.

