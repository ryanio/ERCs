---
title: 3D Models in Token Metadata
description: Adds a model_3d field to token metadata that lists a token's 3D model files by format, such as glTF, USDZ, and VRM.
author: Ryan Ghods (@ryanio)
discussions-to: https://ethereum-magicians.org/t/erc-3d-models-in-token-metadata/00000
status: Draft
type: Standards Track
category: ERC
created: 2026-10-26
---

<!-- Add `requires` with the Token Metadata JSON Schema's number once it has one. -->

## Abstract

This proposal extends the [Token Metadata JSON Schema](./erc-draft_token_metadata_json.md) with a `model_3d` field that lists a token's 3D model files and their formats, so a client can pick the file for its task: a web viewer, AR, an avatar, or a 3D printer. `image` and `animation_url` keep a rendering and a glTF binary for clients that do not implement it.

## Motivation

`animation_url` holds one file in one format, usually glTF. 3D tokens are also used in places that need other formats: USDZ for AR on Apple platforms, VRM for avatars, FBX for game engines, STL and 3MF for printing. Creators who publish these today put them in fields of their own, where clients cannot find them. `animation_url` also does not say what it holds, so a client has to download a file to learn its format and size.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

Terms and URI rules are those of the Token Metadata JSON Schema.

### `model_3d`

`model_3d` is a top-level object with one REQUIRED field, `files`: an array of at least one file object, in order of preference.

| Field | Type | Summary |
| --- | --- | --- |
| `uri` | URI | Location of the file. REQUIRED. |
| `format` | string | Format token from [Formats](#formats). REQUIRED. |
| `version` | string | Version of the format. |
| `bytes` | integer | Size of the file in bytes. |
| `sha256` | string | SHA-256 digest, as 64 lowercase hexadecimal digits. |
| `unit` | string | Length unit: `mm`, `cm`, `m`, or `in`. |

Clients SHOULD use the first entry whose format and version they support for the task at hand, and SHOULD move to the next entry when a file fails to load. Clients MUST skip entries whose `format` or `version` they do not support.

### Formats

| `format` | Format | Media type |
| --- | --- | --- |
| `glb` | glTF 2.0, binary | `model/gltf-binary` |
| `gltf` | glTF 2.0, JSON | `model/gltf+json` |
| `vrm` | VRM avatar (glTF 2.0 binary) | `model/gltf-binary` |
| `usdz` | Universal Scene Description, zip | `model/vnd.usdz+zip` |
| `fbx` | Autodesk FBX | none |
| `obj` | Wavefront OBJ | `model/obj` |
| `stl` | Stereolithography | `model/stl` |
| `3mf` | 3D Manufacturing Format | `model/3mf` |
| `ply` | Polygon File Format | none |
| `spz` | Compressed Gaussian splats | none |

Producers MAY use other tokens. A token SHOULD be the format's usual file extension, in lowercase and without the dot.

`glb`, `gltf`, and `vrm` files MUST be glTF 2.0 and MUST list every glTF extension they need to render in `extensionsRequired`. For `vrm`, `version` SHOULD be `"0.x"` for files using the `VRM` extension and `"1.0"` for files using `VRMC_vrm`. When `version` is absent, clients MAY read it from the file.

Clients MUST resolve the files that `gltf` and `obj` files refer to against the referring file's URI, and MUST NOT follow absolute references other than `data` URIs. Producers SHOULD prefer self-contained formats, and SHOULD keep a `gltf` or `obj` file and the files it refers to in one directory.

For `stl`, `obj`, and `ply`, which do not record units, producers SHOULD set `unit`. Clients that print an `stl` file without `unit` SHOULD assume millimeters.

Clients MAY use `bytes` to decide whether to download a file. Clients that check `sha256` MUST NOT use a file whose digest does not match. Clients MUST NOT run code embedded in or referenced by a model file.

### Fallback

A document with `model_3d` MUST set `image` to a rendering of the model, and SHOULD set `animation_url` to the first `glb` entry. Clients that implement this proposal SHOULD use `model_3d` in place of a 3D model in `animation_url`, and MUST fall back to `animation_url` and then `image` when no entry is usable.

### JSON Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Token Metadata 3D Models",
  "type": "object",
  "properties": {
    "model_3d": {
      "type": "object",
      "required": ["files"],
      "properties": {
        "files": { "type": "array", "minItems": 1, "items": { "$ref": "#/$defs/file" } }
      }
    }
  },
  "$defs": {
    "file": {
      "type": "object",
      "required": ["uri", "format"],
      "properties": {
        "uri": { "type": "string", "format": "uri" },
        "format": { "type": "string", "pattern": "^[a-z0-9][a-z0-9_.+-]*$" },
        "version": { "type": "string" },
        "bytes": { "type": "integer", "minimum": 0 },
        "sha256": { "type": "string", "pattern": "^[0-9a-f]{64}$" },
        "unit": { "type": "string", "enum": ["mm", "cm", "m", "in"] }
      }
    }
  }
}
```

### Example

```json
{
  "name": "Dave Starbelly",
  "image": "ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi/3.png",
  "animation_url": "ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi/3.glb",
  "model_3d": {
    "files": [
      { "uri": "ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi/3.glb", "format": "glb", "bytes": 4823104 },
      { "uri": "ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi/3.usdz", "format": "usdz", "bytes": 6291456 },
      { "uri": "ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi/3.vrm", "format": "vrm", "version": "1.0" },
      {
        "uri": "https://example.com/creatures/3.stl",
        "format": "stl",
        "unit": "mm",
        "sha256": "50d858e0985ecc7f60418aaf0cc5ab587f42c2570a884095a9e8ccacd0f6545c"
      }
    ]
  }
}
```

## Rationale

- `files` is a list, not a map keyed by format. A token can have several files in one format, such as VRM 0.x and 1.0. Each file carries its own size and digest, the order states preference, and tokens such as `3mf` make poor field names.
- Formats are tokens, not media types, because a media type cannot tell VRM from plain glTF and FBX, PLY, and SPZ have none.
- The format list is open, because 3D formats, Gaussian splats especially, change faster than proposals. The table fixes names for common formats so producers do not write `glb` in one document and `gltf-binary` in another.
- `glb` is the fallback because clients that render 3D from `animation_url` already load glTF, so clients with and without this proposal show the same model.
- Deployed collections already publish 3D files in fields of their own. Meebits uses a top-level `vrm` field, and in an October 2026 survey of 708 contracts, ten used an `mml` field for a Metaverse Markup Language (MML) scene. Both map to format tokens.
- Viewer hints, levels of detail, and license terms are left to later revisions, which can add fields that clients of this version ignore.

## Backwards Compatibility

Clients that do not implement this proposal ignore `model_3d` and show `animation_url` or `image`.

## Test Cases

| Client | `files`, in order | Result |
| --- | --- | --- |
| Viewer that supports `glb` | `usdz`, `glb` | Uses `glb`. |
| AR viewer that supports `usdz` | `glb`, `usdz` | Uses `usdz`. |
| Avatar app that supports only VRM 1.0 | `vrm` 0.x, `vrm` 1.0 | Uses the second entry. |
| Viewer that supports `glb` | `blend`, `glb` | Skips `blend`, uses `glb`. |
| Viewer that supports `glb`, first file fails | `glb`, `glb` | Uses the second entry. |
| Viewer that supports `glb` | `fbx` | Falls back to `animation_url`, then `image`. |
| Client that checks digests, digest mismatch | `glb` | Falls back to `animation_url`, then `image`. |
| Printer, no `unit` | `stl` | Reads millimeters. |

## Security Considerations

- Model parsers are complex native code with a history of memory corruption bugs. Clients should parse model files in a sandbox or with memory-safe code.
- Compressed geometry, archives, and large textures can expand far beyond their download size. Clients should check `bytes`, enforce the real size while downloading, and limit vertex counts and texture dimensions.
- Absolute references inside model files could reach tracking servers or internal addresses. The Specification allows only relative and `data` references, and the base proposal's fetching rules apply to every file.
- `model_3d`, `animation_url`, and `image` can show different models, and clients cannot verify that they match. Content-addressed URIs at least keep them from changing later.
- Holding a token does not grant a license to use its model elsewhere. This proposal does not describe license terms.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
