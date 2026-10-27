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

This proposal extends the [Token Metadata JSON Schema](./erc-draft_token_metadata_json.md) with a `model_3d` field that lists a token's 3D model files. Each entry gives a file's URI and format, such as glTF binary, USDZ, VRM, or STL, so that a client can choose the file that fits what it does: show the model in a viewer, place it in augmented reality, load it as an avatar, or send it to a 3D printer. Documents that use the field keep a rendered image in `image` and a glTF binary in `animation_url`, so clients that do not implement this proposal still show the token.

## Motivation

The base metadata format has one place for 3D content: `animation_url`, which clients commonly accept as glTF. One URI in one format does not serve the places 3D tokens end up. AR on Apple platforms and visionOS uses USDZ. Avatar platforms load VRM. Game engines import FBX. 3D printers slice STL and 3MF. Gaussian splat scenes use formats of their own. Creators who produce several of these today put the extra files in `properties`, in fields of their own invention, or on a website, where no client can find them reliably.

`animation_url` also does not say what it contains, so a client has to fetch the file to learn that it is a 3D model and how large it is. A list that names each file's format and size lets a client decide what to download before downloading it.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

This proposal is an extension of the Token Metadata JSON Schema. The terms *metadata document*, *producer*, and *client* have the meanings defined there, and every URI in this proposal follows its rules for resolving and fetching URIs.

### The `model_3d` field

`model_3d` is a top-level field of the metadata document. Its value is an object with these fields:

| Field | Type | Summary |
| --- | --- | --- |
| `files` | array | The token's 3D model files, in order of preference. REQUIRED. |

`files` MUST contain at least one entry. Each entry is an object with these fields:

| Field | Type | Summary |
| --- | --- | --- |
| `uri` | URI | Location of the file. REQUIRED. |
| `format` | string | Format of the file, as described in [Formats](#formats). REQUIRED. |
| `version` | string | Version of the format, for formats with more than one version in use. |
| `bytes` | integer | Size of the file in bytes. |
| `sha256` | string | SHA-256 digest of the file, as 64 lowercase hexadecimal digits. |
| `unit` | string | Length unit, for formats that do not record one. |

Producers SHOULD list files in order of preference. Clients SHOULD use the first entry whose format and version they support for the task at hand. For example, a client with an inline viewer would use the first `glb` entry, and the same client offering AR on an Apple device would use the first `usdz` entry.

### Formats

`format` is a lowercase token. This proposal defines the following tokens:

| `format` | Format | Media type | Typical use |
| --- | --- | --- | --- |
| `glb` | glTF 2.0, binary | `model/gltf-binary` | Web and mobile viewers, AR on Android, game engines |
| `gltf` | glTF 2.0, JSON | `model/gltf+json` | The same as `glb`, with buffers and textures in separate files |
| `vrm` | VRM, a glTF 2.0 binary with humanoid avatar extensions | `model/gltf-binary` | Avatars in games and virtual worlds |
| `usdz` | Universal Scene Description, zip package | `model/vnd.usdz+zip` | AR on Apple platforms, visionOS |
| `fbx` | Autodesk FBX | none registered | Game engines and 3D modeling tools |
| `obj` | Wavefront OBJ | `model/obj` | 3D modeling tools, 3D printing |
| `stl` | Stereolithography | `model/stl` | 3D printing |
| `3mf` | 3D Manufacturing Format | `model/3mf` | 3D printing with color and multiple materials |
| `ply` | Polygon File Format | none registered | Point clouds and 3D Gaussian splats |
| `spz` | SPZ compressed Gaussian splats | none registered | 3D Gaussian splats |

Producers MAY use format tokens that this table does not list. Such a token SHOULD be the format's usual file extension in lowercase, without the leading dot. Clients MUST skip entries whose `format` they do not support.

`glb`, `gltf`, and `vrm` files MUST be glTF 2.0. A file that depends on a glTF extension to render, such as one for compressed geometry or Gaussian splats, MUST list it in the file's `extensionsRequired`, as glTF 2.0 already requires. Clients that cannot load a file because of a required extension SHOULD move to the next entry.

`version` holds the version of the format as its specification writes it. For `vrm`, producers SHOULD set `version` to `"0.x"` for files that use the `VRM` glTF extension and to `"1.0"` for files that use the `VRMC_vrm` glTF extension. Clients MUST skip entries whose `version` they do not support. When `version` is absent, clients MAY read the version from the file.

`gltf` and `obj` files refer to other files, such as buffers, textures, and material libraries. Clients MUST resolve those references against the URI of the file that contains them, and MUST NOT follow references that are absolute URIs other than `data` URIs. Producers SHOULD prefer self-contained formats (`glb`, `vrm`, `usdz`, `3mf`). A producer that uses `gltf` or `obj` SHOULD put every file it refers to in the same directory, such as one IPFS directory.

glTF 2.0 measures length in meters, and USDZ, FBX, and 3MF files record their own units. For `stl`, `obj`, and `ply`, which do not, producers SHOULD set `unit` to `mm`, `cm`, `m`, or `in`. Clients that prepare an `stl` file for printing SHOULD treat a missing `unit` as millimeters.

Clients MAY use `bytes` to decide whether to download a file without the user asking, for example on a metered connection. Clients that check `sha256` MUST NOT use a file whose digest does not match.

Clients MUST NOT run code embedded in or referenced by a model file.

### Fallback fields

A metadata document that contains `model_3d`:

- MUST set `image` to a still rendering of the model.
- SHOULD set `animation_url` to a `glb` file, and that file SHOULD be the first `glb` entry in `files`.

Clients that do not implement this proposal then show the same model in `animation_url`, or the rendering in `image`. When `animation_url` holds a 3D model, clients that implement this proposal SHOULD use `model_3d` instead. When `animation_url` holds other media, such as a turntable video, clients MAY show either.

A client that implements this proposal and cannot use any entry in `files` MUST fall back to `animation_url` and then `image`, as the Token Metadata JSON Schema describes.

### JSON Schema

The following JSON Schema (draft 2020-12) describes the `model_3d` field. Where it differs from the text above, the text applies.

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
        "files": {
          "type": "array",
          "minItems": 1,
          "items": { "$ref": "#/$defs/file" },
          "description": "3D model files, in order of preference."
        }
      }
    }
  },
  "$defs": {
    "file": {
      "type": "object",
      "required": ["uri", "format"],
      "properties": {
        "uri": {
          "type": "string",
          "format": "uri",
          "description": "Location of the file."
        },
        "format": {
          "type": "string",
          "pattern": "^[a-z0-9][a-z0-9_.+-]*$",
          "description": "Format token, such as glb, usdz, vrm, or stl."
        },
        "version": {
          "type": "string",
          "description": "Version of the format, such as 1.0 for VRM 1.0."
        },
        "bytes": {
          "type": "integer",
          "minimum": 0,
          "description": "Size of the file in bytes."
        },
        "sha256": {
          "type": "string",
          "pattern": "^[0-9a-f]{64}$",
          "description": "SHA-256 digest of the file."
        },
        "unit": {
          "type": "string",
          "enum": ["mm", "cm", "m", "in"],
          "description": "Length unit, for formats that do not record one."
        }
      }
    }
  }
}
```

### Example

```json
{
  "name": "Dave Starbelly",
  "description": "Friendly OpenSea Creature that enjoys long swims in the ocean.",
  "image": "ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi/3.png",
  "animation_url": "ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi/3.glb",
  "attributes": [
    { "trait_type": "Base", "value": "Starfish" }
  ],
  "model_3d": {
    "files": [
      {
        "uri": "ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi/3.glb",
        "format": "glb",
        "bytes": 4823104
      },
      {
        "uri": "ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi/3.usdz",
        "format": "usdz",
        "bytes": 6291456
      },
      {
        "uri": "ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi/3.vrm",
        "format": "vrm",
        "version": "1.0",
        "bytes": 9437184
      },
      {
        "uri": "ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi/3.3mf",
        "format": "3mf",
        "bytes": 2097152
      },
      {
        "uri": "https://example.com/creatures/3.stl",
        "format": "stl",
        "unit": "mm",
        "bytes": 3145728,
        "sha256": "50d858e0985ecc7f60418aaf0cc5ab587f42c2570a884095a9e8ccacd0f6545c"
      }
    ]
  }
}
```

## Rationale

### A list of files instead of a map of formats

A map keyed by format, such as `{"glb": "...", "usdz": "..."}`, is shorter to write. This proposal uses a list for four reasons:

- A token can have more than one file in the same format. Avatar platforms differ on whether they load VRM 0.x or VRM 1.0, so producers often publish both, and a producer may offer an optimized `glb` for viewers alongside a full-resolution one.
- Each file can carry its own size, digest, version, and unit.
- The order of the list states the producer's preference.
- Format tokens such as `3mf` begin with a digit. The Token Metadata JSON Schema recommends that field names begin with a letter, and a token is fine as a value.

### Format tokens instead of media types

A media type cannot tell a VRM avatar from a plain glTF binary, since both are `model/gltf-binary`, and FBX, PLY, and SPZ have no registered media type at all. Tokens based on file extensions are short, cover every format, and match what creators already call their files. Servers still send media types for the files themselves.

### An open list of formats

3D formats change faster than proposals can. Gaussian splats moved from research papers into shipping viewers within a few years, and their file formats are still settling. Producers can use any lowercase token and clients skip the ones they do not know, so a new format can appear without a new proposal. The table fixes the names of the formats in common use, so that producers do not write `glb` in one document and `gltf-binary` in another.

### glTF binary as the fallback

Clients that render 3D models from `animation_url` today accept glTF, and the Token Metadata JSON Schema lists it among the formats for that field. A `glb` file is a single file in an open, royalty-free format that browsers load through widely used libraries. Pointing `animation_url` at the first `glb` entry means clients with and without this proposal show the same model.

### Deployed 3D fields

A survey of token metadata from 708 Ethereum and Base contracts in October 2026 found that few documents describe 3D files, and those that do each use a field of their own. Meebits publishes each token's VRM avatar in a top-level `vrm` field. Ten contracts publish a Metaverse Markup Language (MML) scene in a top-level `mml` field. One collection puts a glTF binary in `animation_url` and declares `"format": "GLB"` in an `animation_details` object. A client has to know each collection's field name to find these files, and no field says what other formats a token offers. `model_3d` gives these files one place, and its format tokens match the names these collections already use, so `vrm` and `glb` carry over directly and a scene format such as `mml` can be listed without a new proposal.

### What this proposal leaves out

Viewer hints such as camera position, lighting, environment maps, and which animation clip to play; levels of detail; and license terms for using a model in another application are all useful. They differ by viewer and by use, and a later revision can add them as new fields in the `model_3d` object or in file entries, which clients of this version ignore.

## Backwards Compatibility

Clients that do not implement this proposal ignore `model_3d`, as the Token Metadata JSON Schema requires, and show the `glb` in `animation_url` or the rendering in `image`. Documents that keep 3D files in `properties` or in fields of their own are unaffected.

## Test Cases

| Client | `files`, in order | Result |
| --- | --- | --- |
| Inline viewer that supports `glb` | `usdz`, `glb` | Uses the `glb` entry. |
| AR viewer on an Apple device that supports `usdz` | `glb`, `usdz` | Uses the `usdz` entry. |
| Avatar app that supports only VRM 1.0 | `vrm` with version `0.x`, `vrm` with version `1.0` | Uses the second entry. |
| Viewer that supports `glb` | `blend`, `glb` | Skips the unknown `blend` entry and uses `glb`. |
| Viewer that supports `glb`, where the first file fails to load | `glb`, `glb` | Uses the second entry. |
| Viewer that supports `glb` | `fbx` | No usable entry. Shows `animation_url`, then `image`. |
| Slicer preparing a print | `stl` with no `unit` | Reads the file in millimeters. |
| Client that checks digests, where the file's SHA-256 differs from `sha256` | `glb` | Does not use the file. Falls back to `animation_url`, then `image`. |
| Client that does not implement this proposal | any | Shows `animation_url`, then `image`. |

## Security Considerations

### Parsing untrusted files

Model files are complex binary formats with compressed geometry, embedded textures, and zip archives, and most clients parse them with native code. Bugs in 3D file parsers have led to memory corruption and code execution. Clients SHOULD parse model files in a sandboxed process or with memory-safe code, and SHOULD keep their 3D libraries patched.

### Resource exhaustion

Compressed geometry, USDZ and 3MF archives, and large textures can expand far beyond their download size, and a model with enough vertices or large enough textures can exhaust GPU memory. Clients SHOULD check `bytes` before downloading, enforce the actual size while downloading, and limit vertex counts and texture dimensions.

### References to other files

glTF, OBJ, and USD files can refer to other files. An absolute reference could point at a tracking server or, for a client that fetches on a server, at an internal address. The Specification limits clients to relative and `data` references, and the fetching rules of the Token Metadata JSON Schema apply to every file a client retrieves.

### Mismatched files

The files in `model_3d`, the file in `animation_url`, and `image` can show different things, so a client that implements this proposal and one that does not can show a buyer different models for the same token. Clients cannot verify that the files match. Content-addressed URIs at least keep the set from changing after the fact.

### Executable content

The formats in this proposal describe geometry, materials, and animation, and do not contain code. A client that loads models into an engine that can run scripts, such as a game engine, has to make sure nothing in or referenced by a model file runs, as the Specification requires.

### Usage rights

Holding a token does not by itself grant a license to use its model in another game, application, or printed product. This proposal does not describe license terms.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
