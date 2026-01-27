# CIP / EtherNet/IP Protocol

## Overview

CIP (Common Industrial Protocol) over EtherNet/IP is used to communicate with Allen-Bradley PLCs. tentacle-ethernetip handles both reading and writing tags.

## CIP Type Codes

| Type Code | Type | Size |
|-----------|------|------|
| 0xC1 | BOOL | 1 byte |
| 0xC2 | SINT | 1 byte |
| 0xC3 | INT | 2 bytes |
| 0xC4 | DINT | 4 bytes |
| 0xC5 | LINT | 8 bytes |
| 0xC6 | USINT | 1 byte |
| 0xC7 | UINT | 2 bytes |
| 0xC8 | UDINT | 4 bytes |
| 0xC9 | ULINT | 8 bytes |
| 0xCA | REAL | 4 bytes |
| 0xCB | LREAL | 8 bytes |

## Reading Tags

```typescript
// typeCodeToDatatype() converts CIP type codes to string names
const datatype = typeCodeToDatatype(0xCA); // "REAL"
```

## Writing Tags

```typescript
// datatypeToTypeCode() maps datatype string to CIP type code
const typeCode = datatypeToTypeCode("REAL"); // 0xCA

// encodeTagValue() encodes JS value to Uint8Array
const encoded = encodeTagValue(42.5, "REAL");
// Result: Float32 little-endian bytes
```

### Encoding Examples

```typescript
// BOOL
encodeTagValue(true, "BOOL");   // Uint8Array([1])
encodeTagValue(false, "BOOL");  // Uint8Array([0])

// INT (16-bit signed, little-endian)
encodeTagValue(1000, "INT");    // Uint8Array([0xE8, 0x03])

// DINT (32-bit signed, little-endian)
encodeTagValue(100000, "DINT"); // Uint8Array([0xA0, 0x86, 0x01, 0x00])

// REAL (32-bit float, little-endian)
encodeTagValue(3.14, "REAL");   // IEEE 754 float bytes
```

## Connection Types

1. **ForwardOpen (Connected)**: Establishes a connection for multiple operations
   - May fail with status 0x1 on some PLCs
   - Falls back to UCMM automatically

2. **UCMM (Unconnected)**: Single request/response
   - Works on all PLCs
   - Slightly higher overhead per request

## UDT (User Defined Types)

- Template parsing extracts member names and types
- **Important**: Trust actual PLC response typeCode over template
- Alignment can cause issues in template parsing

## Gotchas

- **Datatype trust hierarchy**: PLC response typeCode > cached datatype > template parsing
- **Little-endian**: All multi-byte values are little-endian
- **ForwardOpen failures**: Normal, just falls back to UCMM
- **String values**: Parse to number before encoding (except BOOL which accepts "true"/"false")
