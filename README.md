# Hot-Patch over TCP Demo

## Hotpatch client demo:
Ships with a deliberately broken function (`bad_compute()`) that will  
segfault when called.  Connects to the hotpatch server, receives a  
replacement function as raw bytes, writes them into an executable page,  
installs a detour over bad_compute, then calls it successfully.

### Flow:
1. Call `bad_compute()` - crashes intentionally (writes to null)
- (we skip the first call and just show what WOULD happen)
2. Connect to hotpatch server on HOTPATCH_PORT
3. Receive PatchHeader  (16 bytes)
4. Validate magic
5. Receive payload      (patch_size bytes) into RWX page
6. Install detour:  `bad_compute()` -> patch page
7. Call `bad_compute()` again - now routed through the fix
8. Send `ACK` to server

## Hotpatch server demo:
Holds the CORRECT implementation of `compute()` as compiled Flux.
### Flow:
1. Serializes the fix function's machine code bytes by reading from its own text segment via a function pointer
2. Sends a PatchHeader followed by the raw bytes
3. Waits for the client ACK
The "fix" is just good_compute — the same logic bad_compute was  
supposed to implement but without the null-write bug:  
`good_compute(x) = x * 3;`
The server reads its OWN compiled `good_compute()` bytes out of memory
and ships them to the client.  The client receives real, already-compiled
machine code and executes it directly - no interpretation, no JIT.

### Shared protocol definitions for the hotpatch server/client demo:
Wire format (all fields little-endian):
```
//   struct PatchPacket
//   {
//       u32  magic,        // 0x48505458 "HPTX" — sanity check
//            patch_size;   // number of bytes in the payload
//       u64  target_rva;   // RVA from client image base to patch site
//                          // (0 = use target_addr directly, for demo)
//       byte payload[];    // raw machine code bytes, patch_size long
//   };
```
The client reads the header first (16 bytes), allocates a page,
// receives exactly patch_size bytes into it, then installs the detour.

## Version 2
Uses HMAC SHA256 to verify the server's signature before applying the patch.
