# Specification

The DAT Metadata Standard builds on the existing [CIP-0025](https://github.com/cardano-foundation/CIPs/tree/master/CIP-0025) standard and is divided into three separate entities.

## **1**. Scene

The *scene* token is the part the end user will receive in their wallet. It contains all the information to render the DAT.

### **1.a.** Metadata

The scene token requires a `renderer` property to be present, referencing the token containing the code of the renderer.

Although not mandatory, it's advisable to bundle token-specific properties into the `properties` object to avoid pollution of the token's root namespace. It will help token viewers to locate and render those properties. 

```json{19-22,24-26}
{
  "721": {
    "<policy_id>": {
      "<asset_name>": {
        "name": <string>,
        "description": <string>,

        "image": <uri | array>,
        "mediaType": image/<mime_sub_type>,
        
        "files": [{
          "name": <string>,
          "mediaType": <mime_type>,
          "src": <uri | array>,
          "license": <string>,
          <other_properties>
        }],
        
        "renderer": {
          "main": <asset_name>,
          "arguments": <array>
        },

        "properties": {
          <properties>
        }
      }
    }
  }
}
```

Properties for the *scene* token:
- **`renderer`** (_required_): an object with two properties
  - **`main`** (_required_): the `asset_name` of the renderer token within the
    current `policy_id` (e.g. `my_renderer`)
  - **`arguments`** (_required_): an array with arbitrary values used as
    arguments for the invocation of the renderer (e.g. `[123]`)
- **`properties`** (_optional_): an object with arbitrary key/value pairs describing the token's (unique) properties

::: details
If a DAT uses data from the blockchain to drive its generative algorithm, it's impossible to create a representational preview image before the mint. So, in that case, the preview image for wallets and marketplaces should be treated more as an album cover than a visual representation of the actual artefact.
:::

### **1.b.** Argument directives

Several directives for dynamic arguments can be passed to the renderer. Before rendering a DAT, viewers must resolve any directives by querying the required data form the blockchain and pass the actual values to the renderer.

_Current token_
- `@tx_hash` (`string`): transaction hash of the mint (can for example be used as the seed value for a Sfc32 PRNG)
- `@epoch` (`number`): epoch in which the token was minted
- `@slot` (`number`): slot in which the token was minted
- `@block` (`number`): block in which the token was minted
- `@block_size` (`number`): size of the token's block
- `@block_hash` (`string`): hash of the token's block

_Previously minted token_
- `@tx_hash.previous` (`string | null`): transaction hash
- `@epoch.previous` (`number | null`): epoch in which the token was minted
- `@slot.previous` (`number | null`): slot in which the token was minted
- `@block.previous` (`number | null`): block in which the token was minted
- `@block_size.previous` (`number | null`): size of the token's block
- `@block_hash.previous` (`string | null`): hash of the token's block
- `@arguments.previous` (`array | null`): token's renderer arguments

_Specific token (within the same policy_id)_
- `@tx_hash.asset_name` (`string | null`): transaction hash
- `@epoch.asset_name` (`number | null`): epoch in which the token was minted
- `@slot.asset_name` (`number | null`): slot in which the token was minted
- `@block.asset_name` (`number | null`): block in which the token was minted
- `@block_size.asset_name` (`number | null`): size of the token's block
- `@block_hash.asset_name` (`string | null`): hash of the token's block
- `@arguments.asset_name` (`array | null`): token's renderer arguments

_Current blockchain state_
- `@current_epoch` (`number`): current (latest) epoch
- `@current_slot` (`number`): current (latest) slot
- `@current_block` (`number`): current (latest) minted block
- `@current_block_size` (`number`): size of the current block
- `@current_block_hash` (`string`): hash of the current block

Passing argument directives to the renderer works just like static arguments. For example:

```json
[
  123,
  "@tx_hash",
  "@block",
  "@current_block",
  "@epoch.previous",
  "@arguments.the_perfect_nft_000"
]
```

::: warning
Referencing another token's arguments does not work recursively.
:::

::: info
The list of directives mentioned above is not exhaustive, nor final. Proposals for new directives will be considered and added over time.
:::

## **2**. Renderer

The *renderer* token is always part of the same `policy_id`. It can either be a self-contained program or one with dependencies. Within the same policy, multiple *renderer* tokens can exist, but *scene* tokens can only reference one at a time.

### **2.a.** Metadata

The renderer's code is stored in the `files` property as-is or as a base64-encoded string. The `name` property of the file must match the `asset_name` with the appropriate file extenstion, so viewers can filter out the renderer-related files.

Instructions and/or requirements to reproduce the token can be stored in a file named `instructions`. For browser-based artworks, this should include the latest browser version(s) in which the token works. For projects executed locally, it should be a dependency file for a package manager (see attachment `examples/renderer/instructions.md`).

```json{9,13,15-18}
{
  "721": {
    "<policy_id>": {
      "<asset_name>": {
        "files": [{
          "name": <asset_name>.<extension>,
          "mediaType": <mime_type>,
          "src": <uri | array>,
          "license": <string>,
          <other_properties>
        }],

        "outputType": <mime_type>,

        "dependencies": [{
          "type": <string>,
          <other_properties>
        }]
      }
    }
  }
}
```

Properties for the *renderer* token:
- **`outputType`** (_required_): the mime type of the renderer's output (it's up to the viewer to define the supported formats)
- **`dependencies`** (_optional_): an array of objects with dependency definitions

::: info
While not mandatory, it's advisable to add a **`license`** property to every file in the `files` section. More info on licenses below in section 2.e.
:::

::: tip
The renderer token should be burned after minting to free up the UTxO.
:::

### **2.b.** On-chain dependencies

These are policy-specific on-chain dependencies managed by the creator. They must be minted within the same `policy_id` and are referenced by their `asset_name`.

```json
{
  "type": "onchain",
  "asset_name": <asset_name>
}
```

### **2.c.** Internal dependencies:

These are on-chain dependencies managed internally by the viewer and made available to the *renderer* at runtime. They can be referenced by using the dependency's asset `fingerprint`:

```json
{
  "type": "internal",
  "fingerprint": <asset_fingerprint>
}
```

Or the asset's `policy_id` and `asset_name`:

```json
{
  "type": "internal",
  "policy_id": <policy_id>,
  "asset_name": <asset_name>
}
```

::: tip
Using the `fingerprint` is more concise, but it doesn't tell anything about the referenced asset. Although being more verbose, the second option is more readable as it includes the human-readable `asset_name`. It does not matter which option is used, they both return the same token.
:::

### **2.d.** External dependencies:

These are off-chain dependencies managed by the viewer and made available to the *renderer* at runtime.

```json
{
  "type": "external",
  "name": <library_name>,
  "version": <version_number>
}
```

::: info
The _external_ dependency definitions are not referencing any token on the blockchain, unlike the _onchain_ and _internal_ variants. Their sole purpose is to instruct token viewers which external dependency to load at runtime. More details can be found in the [Examples](/dat-metadata-standard/examples) section.
:::

### **2.e.** License types

For creators, it is recommended to choose a license that aligns with their values and with the purpose of the digital artefact. Any license can be used, but popular licenses are:

- [NFT License 2.0](https://www.nftlicense.org/)
- [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/)
- [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
- [AGPL 3.0](https://www.gnu.org/licenses/agpl-3.0.en.html)

Using [no license](https://choosealicense.com/no-permission/) is also an option to explicitly indicate that no one other than the creator may use the software.

## **3**. Dependency

This section applies to _onchain_ and _internal_ dependencies. Aside from how they are referenced, there's no difference between them.

::: details
What sets them apart is the scope in which they are available and by whom they are managed:
- _onchain_: managed by creators, must be minted under the same `policy_id` as the _renderer_ token referencing them and only available within that same policy
- _internal_: managed by token viewers and globally available to anyone using the DAT Metadata Standard
:::

Dependency tokens can consist of multiple parts if they don't fit into one 16kB transaction. The dependency referenced from the _renderer_ serves as an entrypoint, in turn referencing the additional parts.

::: warning
The number of accepted dependency parts is limited. Token viewers decide how many parts to allow or support. Ten parts should be more than sufficient in most scenarios. 
:::

### **3.a.** Metadata

The code is stored in the **`files`** property as-is or as a base64-encoded string. The `name` property of the file must match the `asset_name`. Similar to the *renderer*, every file can have an individual `license` property.

```json{9,13-15}
{
  "721": {
    "<policy_id>": {
      "<asset_name>": {
        "files": [{
          "name": <asset_name>.<extension>,
          "mediaType": <mime_type>,
          "src": <uri | array>,
          "license": <string | null>,
          <other_properties>
        }],

        "parts": [
          <part_asset_name>
        ]
      }
    }
  }
}
```

Properties for the *dependency* token:
- **`parts`** (_optional_): an array with asset names (e.g. `asset_name_part_2`)

::: info
While not mandatory, it's advisable to add a **`license`** property to each file in the `files` section. More info on licenses in section 2.e.
:::

::: tip
Dependency tokens should be burned after minting to free up the UTxOs.
:::