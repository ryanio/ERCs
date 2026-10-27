---
title: Token Metadata JSON Schema
description: Specifies the JSON that token URIs resolve to, including media fields, traits, and a way for later proposals to extend it.
author: Ryan Ghods (@ryanio)
discussions-to: https://ethereum-magicians.org/t/erc-token-metadata-json-schema/00000
status: Draft
type: Standards Track
category: ERC
created: 2026-10-26
requires: 721, 1155, 4804, 4906, 6909
---

## Abstract

This proposal specifies the JSON document that a token's metadata URI resolves to: the value returned by `tokenURI` for [ERC-721](./eip-721.md) and [ERC-6909](./eip-6909.md) tokens, and by `uri` for [ERC-1155](./eip-1155.md) tokens. It defines the display fields that wallets and marketplaces read (`name`, `description`, `image`, `animation_url`, `external_url`, `background_color`), the `attributes` array of traits with optional display hints, the URI schemes clients resolve, and how clients treat missing or malformed values. It also defines how later proposals add fields under their own top-level names, so that documents using an extension still render in clients that do not implement it. The format is a superset of the metadata schemas in ERC-721 and ERC-1155 and matches what most deployed tokens already publish.

## Motivation

ERC-721 defines three metadata fields: `name`, `description`, and `image`. ERC-1155 adds `decimals`, `properties`, and `localization`. Almost everything else that wallets and marketplaces show for a token, including video, audio, interactive HTML, and the traits used for filtering and rarity, comes from a format OpenSea published as developer documentation in 2018. That format became the working schema for token metadata on Ethereum and other EVM chains, and ecosystems outside Ethereum adopted its attribute shape, including Solana's Metaplex token metadata and Hedera's HIP-412.

Because the format was never specified, it leaves questions open: whether booleans are valid trait values, whether a quoted number is a number, whether dates are in seconds or milliseconds, what an attribute without a `trait_type` means, which URI schemes a client has to resolve, and how to isolate an HTML page in `animation_url`. Each client answers these its own way. Creators test against one marketplace and find out later how the others behave.

Proposals that add metadata fields, such as [ERC-5625](./eip-5625.md), extend the ERC-721 schema directly because there is no base document to require. Nothing tells them how to name new fields, how to avoid colliding with other proposals, or what clients that do not implement them should show. A formal base gives future extensions, such as a structured description of 3D model files, a document to build on and rules for staying compatible with the clients that exist today.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Definitions

- A *metadata URI* is the string returned by `tokenURI(uint256)` in ERC-721 or ERC-6909, or by `uri(uint256)` in ERC-1155.
- A *metadata document* is the JSON object that a metadata URI resolves to.
- A *producer* is the contract or creator that publishes metadata documents.
- A *client* is software that reads metadata documents, such as a wallet, marketplace, or indexer.
- An *extension* is a proposal that adds fields to the metadata document, as described in [Extensions](#extensions).

### Resolving URIs

These rules apply to the metadata URI and to every URI-valued field in a metadata document.

Clients MUST support the following URI forms:

| Form | Example | Resolution |
| --- | --- | --- |
| `https` | `https://example.com/metadata/1.json` | HTTP GET. |
| `ipfs` | `ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi/1.json` | Retrieve the content identifier, and the path if present, from IPFS directly or through a gateway. |
| `ar` | `ar://jK9sR4OrYvODj7PD3czIAyNJalub0-vdV_JAg1NqQ-o` | Retrieve the transaction data, and the path if present, from Arweave. |
| `data` | `data:application/json;base64,eyJuYW1lIjoiRXhhbXBsZSJ9` | Decode the payload as described in [RFC 2397](https://www.rfc-editor.org/rfc/rfc2397). |

Producers MUST use absolute URIs. Producers SHOULD use a content-addressed form (`ipfs`, `ar`, or `data`) for metadata and media that are not meant to change, and SHOULD NOT use `http`.

In an `ipfs` URI, the content identifier comes directly after `ipfs://`. Producers MUST NOT write `ipfs://ipfs/<cid>`, and SHOULD NOT write an IPFS gateway URL such as `https://gateway.example/ipfs/<cid>` in place of an `ipfs` URI, since the gateway can go offline. Clients MAY fetch `ipfs` and `ar` content through any gateway.

A `data` URI that carries a metadata document SHOULD use the `data:application/json;base64,` prefix. Many deployed contracts return unencoded JSON instead, as in `data:application/json;utf8,{"name":"Example"}`. Clients MUST accept base64 payloads and SHOULD accept unencoded payloads with or without a `utf8` or `charset=utf-8` parameter, parsing the payload as JSON as written and, if that fails, after percent-decoding it.

Clients MAY support other schemes, such as `ipns` or the `web3` scheme defined in [ERC-4804](./eip-4804.md). Clients MUST NOT resolve schemes that run code or read local resources, such as `javascript` or `file`.

For ERC-1155 tokens, clients MUST apply the `{id}` substitution that ERC-1155 defines, both to the metadata URI and to string values in the metadata document.

### Fetching

A metadata document MUST be a JSON object as defined in [RFC 8259](https://www.rfc-editor.org/rfc/rfc8259), encoded as UTF-8. Clients MUST treat a document whose top-level value is not an object as absent.

Servers that host metadata documents or media over `https` SHOULD send an accurate `Content-Type` header and SHOULD send `Access-Control-Allow-Origin: *`, so that browser-based clients can read them directly. Clients SHOULD attempt to parse a metadata document as JSON whatever its declared content type.

A client that fetches metadata documents or media from its own servers, such as an indexer, MUST NOT connect to loopback, private, or link-local addresses. It MUST apply this check to the address each hostname resolves to and to every redirect.

### Fields

| Field | Type | Summary |
| --- | --- | --- |
| `name` | string | Name of the token. |
| `description` | string | Description of the token, which MAY contain Markdown. |
| `image` | URI | Still image of the token. |
| `animation_url` | URI | Video, audio, 3D model, or HTML page for the token. |
| `external_url` | URI | Page about the token on the producer's site. |
| `background_color` | string | Color behind the image, as six hexadecimal digits. |
| `attributes` | array | Traits of the token. |

Every field is OPTIONAL. Producers SHOULD include `name` and `image`.

Producers SHOULD omit a field rather than set it to `null` or to an empty string. Clients MUST treat a field whose value is `null` or an empty string as absent. When a field has the wrong type, clients SHOULD ignore that field and process the rest of the document.

Clients MUST ignore fields they do not recognize. A document that contains fields this proposal does not define is still conformant.

Clients MAY truncate or drop values that exceed their own size limits.

#### `name`

The name of the token as plain text, such as `"Dave Starbelly"` or `"Creature #3"`. Clients MUST NOT interpret Markdown or HTML in it.

#### `description`

A human-readable description of the token. It MAY contain Markdown as defined by CommonMark. Clients that render Markdown MUST NOT render raw HTML contained in it and SHOULD NOT load remote images it references. Clients that do not render Markdown SHOULD show it as plain text.

#### `image`

The URI of a still image that represents the token. Clients use it in lists, grids, and previews, and as the poster or fallback for `animation_url`.

Producers SHOULD use an image format that browsers display natively, such as PNG, JPEG, GIF, WebP, AVIF, or SVG. Animated GIF and WebP images are allowed. Raster images SHOULD be large enough to display sharply at full screen on high-density displays.

Clients MUST display SVG images in a way that does not run scripts they contain, for example as an image element rather than inline markup, or by rasterizing them.

#### `animation_url`

The URI of a richer form of the token: a video, an audio file, a 3D model, or an HTML page. A client that can render the referenced media SHOULD show it in the token's detail view in place of `image`, and MUST fall back to `image` when it cannot. When `animation_url` is audio, clients SHOULD show `image` as its cover art.

Producers SHOULD use one of these formats, which clients that render `animation_url` commonly support:

| Kind | Formats | Media types |
| --- | --- | --- |
| Video | MP4, M4V, WebM, Ogg | `video/mp4`, `video/x-m4v`, `video/webm`, `video/ogg` |
| Audio | MP3, WAV, Ogg | `audio/mpeg`, `audio/wav`, `audio/ogg` |
| 3D model | glTF 2.0, as `.gltf` or `.glb` | `model/gltf+json`, `model/gltf-binary` |
| Interactive | HTML | `text/html` |

Clients SHOULD determine the media type from the `Content-Type` response header or from the media type of a `data` URI, and MAY fall back to the file extension. Producers SHOULD serve accurate media types.

A client that renders an HTML page:

- MUST isolate the page from the client's own origin, for example in an `iframe` whose `sandbox` attribute grants `allow-scripts` but not `allow-same-origin`, or by serving it from a separate origin.
- MUST NOT expose a wallet provider or any signing interface to the page.
- SHOULD load the page so that relative references in it resolve against its own location, so that a page at `ipfs://<cid>/index.html` can load `ipfs://<cid>/sketch.js` as `sketch.js`.

#### `external_url`

The URI of a page about the token on the producer's site. Clients MAY link to it. Producers SHOULD use `https`.

#### `background_color`

An sRGB color written as six hexadecimal digits without a leading `#`, such as `"E0F2FE"`. Letters are case-insensitive. Clients that place the image on a background SHOULD use this color. Clients SHOULD ignore a value in any other format, and MAY first remove a leading `#`.

#### `attributes`

An array of objects that describe the token's traits. Clients use them for display, for filtering, and for rarity calculations within a collection. Each object has these fields:

| Field | Type | Summary |
| --- | --- | --- |
| `trait_type` | string | Name of the trait, such as `"Background"`. |
| `value` | string or number | Value of the trait. REQUIRED. |
| `display_type` | string | How clients present the value. |
| `max_value` | number | Upper bound of a numeric value. |

`value` MUST be a string or a number. Clients SHOULD ignore an attribute whose `value` is absent, `null`, an empty string, an object, or an array. Clients MAY convert a boolean `value` to the string `"true"` or `"false"`.

A string `value` is a string even when it looks like a number: `"5"` is a string trait and `5` is a numeric trait. Producers MUST write numeric traits as JSON numbers.

`trait_type` SHOULD be present. Clients MAY show an attribute without one under a generic label.

Clients that group attributes across a collection SHOULD match `trait_type` and string values exactly after removing leading and trailing whitespace. Producers SHOULD spell and capitalize each trait type and value the same way on every token in a collection.

A document MAY contain more than one attribute with the same `trait_type` to give a trait several values. Clients SHOULD keep each of them.

Producers SHOULD list attributes in the order they want them shown. Clients MAY reorder them.

`display_type` takes one of these values:

| `display_type` | `value` | Meaning |
| --- | --- | --- |
| absent | string | A categorical trait, such as `"Blue"`. |
| absent | number | A numeric trait. Clients MAY show it as a bar or ranking relative to `max_value`. |
| `string` | string | A categorical trait, the same as omitting `display_type`. |
| `text` | string | The same as `string`. Deployed documents use both names, so clients MUST accept `text`. Producers SHOULD write `string`. |
| `number` | number | A numeric stat, shown as a number. |
| `boost_number` | number | A numeric bonus, shown with its sign, such as `+40`. |
| `boost_percentage` | number | A percentage bonus, shown with its sign and a percent sign. `10` means `+10%`. |
| `date` | number | A point in time, as Unix time in seconds. `1546360800` is `2019-01-01T16:40:00Z`. |

When `display_type` is `number`, `boost_number`, `boost_percentage`, or `date`, `value` MUST be a number. Clients MAY accept a string `value` that is a decimal number in that case.

A `date` value SHOULD be an integer.

Clients MUST treat a `display_type` they do not recognize as absent. Extensions MAY define new `display_type` values.

`max_value` applies only to numeric traits. When present it MUST be a number greater than or equal to `value`. When it is absent, clients MAY derive a maximum from the values of the same trait on other tokens in the collection.

#### Fields from ERC-1155

`decimals`, `properties`, and `localization` have the meaning ERC-1155 gives them, in documents for tokens of any standard. Producers SHOULD publish traits in `attributes` rather than in `properties`.

#### Legacy fields

Clients MAY support the following fields, which appear in deployed documents. Producers SHOULD NOT use them in new documents.

- `image_data` is raw SVG markup. Clients MAY show it as the image when `image` is absent. Producers SHOULD use `image` with a `data:image/svg+xml` URI instead.
- `youtube_url` is the URL of a YouTube video. Clients MAY use it when `animation_url` is absent.

#### Compatibility with deployed metadata

Clients MAY accept the following forms, which appear in deployed documents but do not conform to this proposal. Producers MUST NOT use them.

- `attributes` written as an object that maps trait types to values, such as `{"Background": "Blue"}`.
- An attributes array named `traits` or `properties` instead of `attributes`.
- An image URI in a field named `image_url` instead of `image`.
- `ipfs://ipfs/<cid>` URIs, which clients can read as `ipfs://<cid>`.
- `date` values written in milliseconds. A value of `1000000000000` or more is after the year 33000 when read as seconds and after September 2001 when read as milliseconds, so clients MAY read such values as milliseconds. Smaller values are seconds, including `253402300800`, the start of the year 10000, which some contracts write to mean a date that never arrives.

### Extensions

An extension is a proposal that adds fields to the metadata document. Extensions follow these rules:

1. An extension MUST list this proposal in its `requires` header.
2. An extension MUST name every top-level field it defines. It SHOULD define a single top-level field whose value is a JSON object and put its data inside that object, so that later revisions can add properties without claiming more top-level names.
3. Field names SHOULD contain only lowercase ASCII letters, digits, and underscores, and SHOULD begin with a letter.
4. An extension MUST NOT change the meaning or type of a field defined by this proposal or by another extension. An extension MAY define new `display_type` values and new fields in attribute objects.
5. A document that uses an extension MUST still conform to this proposal. An extension SHOULD say which base fields producers fill as a fallback, so that clients that do not implement it still show the token. For example, an extension that describes 3D models could require `image` to hold a rendered preview and `animation_url` to hold a glTF binary.
6. Clients that do not implement an extension MUST ignore its fields. Clients that implement it SHOULD prefer its data over the fallback fields when both are present.

Producers MAY include fields that neither this proposal nor any extension defines. Such fields SHOULD use names unlikely to collide with a future extension, such as names that include the project's name.

### Metadata updates

Producers MAY change a token's metadata document. When they do, ERC-721 contracts SHOULD emit `MetadataUpdate` or `BatchMetadataUpdate` as defined in [ERC-4906](./eip-4906.md), and ERC-1155 contracts SHOULD emit the `URI` event defined in ERC-1155. Clients that cache metadata documents SHOULD refetch the affected tokens when they observe these events, and MAY also refetch on their own schedule.

### JSON Schema

The following JSON Schema (draft 2020-12) describes a conformant metadata document. Where it differs from the text above, the text applies.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Token Metadata",
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "description": "Name of the token, as plain text."
    },
    "description": {
      "type": "string",
      "description": "Description of the token. May contain CommonMark Markdown."
    },
    "image": {
      "type": "string",
      "format": "uri",
      "description": "URI of a still image of the token."
    },
    "animation_url": {
      "type": "string",
      "format": "uri",
      "description": "URI of a video, audio file, 3D model, or HTML page for the token."
    },
    "external_url": {
      "type": "string",
      "format": "uri",
      "description": "URI of a page about the token on the producer's site."
    },
    "background_color": {
      "type": "string",
      "pattern": "^[0-9A-Fa-f]{6}$",
      "description": "Background color as six hexadecimal digits, without a leading #."
    },
    "attributes": {
      "type": "array",
      "items": { "$ref": "#/$defs/attribute" },
      "description": "Traits of the token."
    },
    "decimals": {
      "type": "integer",
      "minimum": 0,
      "description": "As defined in ERC-1155."
    },
    "properties": {
      "type": "object",
      "description": "As defined in ERC-1155."
    },
    "localization": {
      "type": "object",
      "required": ["uri", "default", "locales"],
      "properties": {
        "uri": { "type": "string" },
        "default": { "type": "string" },
        "locales": { "type": "array", "items": { "type": "string" } }
      },
      "description": "As defined in ERC-1155."
    },
    "image_data": {
      "type": "string",
      "deprecated": true,
      "description": "Legacy. Raw SVG markup, used when image is absent."
    },
    "youtube_url": {
      "type": "string",
      "format": "uri",
      "deprecated": true,
      "description": "Legacy. YouTube video URL, used when animation_url is absent."
    }
  },
  "$defs": {
    "attribute": {
      "type": "object",
      "required": ["value"],
      "properties": {
        "trait_type": { "type": "string" },
        "value": { "type": ["string", "number"] },
        "display_type": { "type": "string" },
        "max_value": { "type": "number" }
      },
      "if": {
        "required": ["display_type"],
        "properties": {
          "display_type": {
            "enum": ["number", "boost_number", "boost_percentage", "date"]
          }
        }
      },
      "then": {
        "properties": { "value": { "type": "number" } }
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
  "external_url": "https://example.com/creatures/3",
  "background_color": "E0F2FE",
  "attributes": [
    { "trait_type": "Base", "value": "Starfish" },
    { "trait_type": "Eyes", "value": "Big" },
    { "trait_type": "Level", "value": 5, "max_value": 10 },
    { "display_type": "number", "trait_type": "Generation", "value": 2 },
    { "display_type": "boost_number", "trait_type": "Aqua Power", "value": 40 },
    { "display_type": "boost_percentage", "trait_type": "Stamina Increase", "value": 10 },
    { "display_type": "date", "trait_type": "Birthday", "value": 1546360800 }
  ]
}
```

## Rationale

### Describing the deployed format

The goal is to specify the format that producers already publish and clients already read, not to design a new one. Field names stay as deployed, including the awkward ones. `animation_url` holds video, audio, 3D models, and HTML, not only animations, and the token-level `external_url` differs from the `external_link` field in the contract-level metadata of [ERC-7572](./eip-7572.md). Renaming either would break existing documents and give clients nothing new.

### No required fields

ERC-721 marks no metadata field as required, and deployed documents omit fields often, so clients have to handle absence regardless. Recommending `name` and `image` tells producers what clients need without declaring a large share of existing documents invalid.

### Attributes as an array

An array of objects lets a trait repeat, keeps the producer's order, and carries a display hint for each trait. An object keyed by trait type can do none of these. The array is also the form that most deployed documents use and that formats on other chains copied.

### `display_type` as a hint

`display_type` tells clients how to present a value, and the JSON type of `value` says whether it is a string or a number. Treating an unknown `display_type` as absent lets a producer use a display type that only some clients understand without breaking the others, and lets extensions add display types later.

### Dates in seconds

Unix time in seconds matches `block.timestamp`, so contracts that build metadata onchain can write dates without converting them.

Some deployed documents write dates in milliseconds, and some write `253402300800`, the start of the year 10000 in seconds, to mean a date that never arrives. A cutoff of `1000000000000` separates the two. Every millisecond date after September 2001 is above it, and no date in seconds reaches it before the year 33000. A lower cutoff of `100000000000` would read the year-10000 value as a date in 1978.

### `ipfs` URIs instead of gateway URLs

A gateway URL ties a document to one operator's server. In a sample of token URIs from 708 Ethereum and Base contracts taken in October 2026, 113 of the 204 written as IPFS gateway URLs failed to load as written, and every sampled token failed for 18 of the 30 contracts that used them. Several public gateways now serve only browsers, or answer automated requests with a bot challenge. An `ipfs` URI names the content rather than a server, so a client can fetch it through any gateway or node that has it.

### `text` as a display type

`text` is not in the format OpenSea documented, but more deployed collections in the same sample use `display_type: "text"` than `"string"`. Clients that treat unknown display types as absent already show these traits as strings. Listing `text` as another name for `string` makes that behavior explicit.

### Extensions as top-level fields

Clients already detect optional data by checking whether a field is present. An extension that owns one top-level object fits that pattern. There is no version number to coordinate and no registry to consult, and a client can implement one extension without the others. Two alternatives were considered:

- A list in the document of the proposals it implements, similar to the `interop` object in [ERC-1046](./eip-1046.md). This repeats what the presence of the fields already says, and the list and the fields can disagree.
- A `$schema` or version field. Clients would need to understand each version to read any of them, and producers that combine extensions would need a combined schema.

Field names that begin with a letter work as identifiers in most programming languages and in the schema languages that clients use to expose metadata, such as GraphQL and Protocol Buffers, neither of which allows a field name to begin with a digit.

### No contract interface

This proposal defines no contract interface and no [ERC-165](./eip-165.md) identifier. The document lives offchain or in a `data` URI, a contract that already returns conformant metadata needs no changes, and a client learns everything an interface could tell it by reading the document.

### Related proposals

ERC-7572 covers contract-level metadata returned by `contractURI()`, and this proposal covers the per-token document. [ERC-7496](./eip-7496.md) covers traits stored onchain, which contracts and marketplace orders can verify at the time of a sale. Attributes in this proposal are display data that a producer can change, so a protocol that settles on a trait's value needs an onchain source such as ERC-7496. ERC-1046 defines `tokenURI()` for [ERC-20](./eip-20.md) tokens with its own schema, which this proposal leaves unchanged.

## Backwards Compatibility

ERC-721 and ERC-1155 each define a subset of the fields in this proposal with the same meanings, so every document that conforms to either of their metadata schemas conforms to this proposal. Documents that follow the metadata format in OpenSea's developer documentation conform as well.

Deployed documents that use the forms listed in [Compatibility with deployed metadata](#compatibility-with-deployed-metadata) do not conform, and clients that accept those forms can still read them. The legacy fields `image_data` and `youtube_url` keep the meaning they have in deployed documents.

## Test Cases

### Resolving metadata URIs

| Metadata URI | Document |
| --- | --- |
| `data:application/json;base64,eyJuYW1lIjoiRXhhbXBsZSJ9` | `{"name":"Example"}` |
| `data:application/json;utf8,{"name":"Example"}` | `{"name":"Example"}` |
| `ipfs://ipfs/bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi/1.json` | Not conformant. Clients that accept it read it as `ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi/1.json`. |

For an ERC-1155 token with ID `314592` and metadata URI `ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi/{id}.json`, clients fetch `ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi/000000000000000000000000000000000000000000000000000000000004cce0.json`.

### Reading fields

| Document | Result |
| --- | --- |
| `{"name": "Example", "image": 42}` | `name` is `Example`. `image` has the wrong type and is ignored. |
| `{"name": "Example", "description": null}` | `description` is absent. |
| `{"name": "Example", "background_color": "#E0F2FE"}` | Not conformant. A client that removes the leading `#` reads the color as `E0F2FE`. |
| `{"name": "Example", "glow": {"color": "gold"}}` | `glow` is unknown and ignored. The document conforms. |

### Reading attributes

| Attribute | Result |
| --- | --- |
| `{"trait_type": "Eyes", "value": "Big"}` | String trait `Eyes` with value `Big`. |
| `{"trait_type": " Eyes ", "value": "Big "}` | The same trait and value as the previous row. |
| `{"value": "Happy"}` | String trait with value `Happy` and no trait type. |
| `{"trait_type": "Level", "value": 5, "max_value": 10}` | Numeric trait `Level` with value 5 out of 10. |
| `{"trait_type": "Level", "value": "5"}` | String trait `Level` with value `5`. |
| `{"display_type": "boost_percentage", "trait_type": "Stamina Increase", "value": 10}` | Numeric trait shown as `+10%`. |
| `{"display_type": "date", "trait_type": "Birthday", "value": 1546360800}` | Date trait with value `2019-01-01T16:40:00Z`. |
| `{"display_type": "text", "trait_type": "Faction", "value": "Red"}` | String trait `Faction` with value `Red`. |
| `{"display_type": "date", "trait_type": "Registered", "value": 1694173211000}` | Date trait with value `2023-09-08T11:40:11Z` for clients that read large values as milliseconds. |
| `{"display_type": "date", "trait_type": "Expires", "value": 253402300800}` | Date trait with value `10000-01-01T00:00:00Z`. |
| `{"display_type": "sparkle", "trait_type": "Mood", "value": "Calm"}` | `sparkle` is unknown, so this is string trait `Mood` with value `Calm`. |
| `{"trait_type": "Mood", "value": null}` | Ignored. |
| `{"trait_type": "Hat", "value": ["Cap"]}` | Ignored. |

A document whose attributes are `{"trait_type": "Accessory", "value": "Scarf"}` and `{"trait_type": "Accessory", "value": "Glasses"}` has one trait, `Accessory`, with the values `Scarf` and `Glasses`.

## Security Considerations

### Active content

HTML pages in `animation_url` and SVG images can run scripts. A client that renders them in its own origin hands the producer its cookies, its storage, and any connected wallet. The Specification requires clients to isolate HTML pages, to withhold wallet providers from them, and to display SVG without running its scripts. Clients that render HTML SHOULD also block popups and top-level navigation from the page.

### Links and Markdown

`description` and `external_url` come from the producer, who can be an attacker. A Markdown link or `external_url` can lead to a phishing page that imitates the client. Clients SHOULD follow only `https` links from these fields and SHOULD show the destination's domain before leaving.

### Server-side fetching

Indexers fetch URIs that anyone who deploys a contract can choose. Without the address checks in [Fetching](#fetching), a metadata URI or a redirect could reach services on the indexer's internal network, such as a cloud provider's instance metadata endpoint.

### Resource exhaustion

Documents and media can be very large, very slow, deeply nested, or compressed to expand on download. Clients SHOULD limit response size, nesting depth, request time, and the number of redirects they follow.

### Tracking and cloaking

A producer that serves documents or media from its own `https` server sees the IP address of every client that fetches from it directly. It can also serve different content to different requesters, for example a harmless image to a marketplace's moderation crawler and a different one to users' browsers. Clients SHOULD fetch through their own cache or proxy and show users the cached copy.

### Mutable metadata

A document served over `https` can change at any time, including between a buyer's decision and settlement. This matters for offers that target any token with a given trait: a seller can change a token's attributes to match an offer, fill it, and change them back. A content-addressed metadata URI can only change if the contract returns a new URI, which is visible onchain. Clients SHOULD show whether a token's metadata is content-addressed, and protocols that settle on a trait's value SHOULD read it from an onchain source such as ERC-7496 rather than from this document.

### Impersonation

Anyone can deploy a token whose name, image, and description copy those of another collection, including names that use visually similar Unicode characters. Clients SHOULD identify tokens by chain, contract address, and token ID, never by metadata.

### Self-reported traits

Rarity and other values computed from attributes are only as reliable as the producer that writes them, and a producer can change them at any time unless the metadata is content-addressed.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
