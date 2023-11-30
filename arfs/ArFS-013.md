# Arweave File System (ArFS)

Status: Draft

Version: 0.13

## Abstract

This document describes data and metadata standards for the Arweave File System.

## Motivation

Because of Arweave's permanent and immutable nature, traditional file structure operations such as renaming and moving files or folders cannot be accomplished by simply updating on-chain data. ArFS works around this by defining an append-only transaction data model based on the metadata tags found in the Arweave [Transaction Headers.](https://docs.arweave.org/developers/server/http-api#transaction-format)

## Specification

### Metadata Format

Metadata stored in any Arweave transaction tag will be defined in the following manner:

```json
{ "name": "Example-Tag", "value": "example-data" }
```

Metadata stored in the Transaction Data Payload will follow JSON formatting like below:

```json
{
    "exampleField": "exampleData"
}
```

fields with a `?` suffix are optional.

```json
{
  "name": "My Project",
  "description": "This is a sample project.",
  "version?": "1.0.0",
  "author?": "John Doe"
}
```

Enumerated field values (those which must adhere to certain values) are defined in the format "value 1 | value 2".

All UUIDs used for Entity-Ids are based on the [Universally Unique Identifier](https://en.wikipedia.org/wiki/Universally_unique_identifier) standard.

There are no requirements to list ArFS tags in any specific order.

### Entity Types

Arweave transactions are composed of transaction headers and data payloads.

ArFS entities, therefore, have their data split between being stored as tags on their transaction header and encoded as JSON and stored as the data of a transaction. In the case of private entities, JSON data and file data payloads are always encrypted according to the protocol processes defined below.

- Drive entities require a single metadata transaction, with standard Drive tags and encoded JSON with secondary metadata.

- Folder entities require a single metadata transaction, with standard Folder tags and an encoded JSON with secondary metadata.

- File entities require a metadata transaction, with standard File tags and an encoded Data JSON with secondary metadata relating to the file.

- File entities also require a second data transaction, which includes a limited set of File tags and the actual file data itself.

- Snapshot entities require a single transaction. which contains a Data JSON with all of the Drive’s rolled up ArFS metadata and standard Snapshot GQL tags that identify the Snapshot.

#### Drive Entity

A drive is the highest level logical grouping of folders and files. All folders and files must be part of a drive, and reference the Drive ID of that drive.

When creating a Drive, a corresponding folder must be created as well. This will act as the root folder of the drive. This separation of drive and folder entity enables features such as folder view queries, renaming, and linking.

```json
ArFS: "0.13"
Cipher?: "AES256-GCM"
Cipher-IV?: "<12 byte initialization vector as Base64>"
Content-Type: "<application/json | application/octet-stream>"
Drive-Id: "<uuid>"
Drive-Privacy: "<public | private>"
Drive-Auth-Mode?: "password"
Entity-Type: "drive"
Unix-Time: "<seconds since unix epoch>"

Data JSON {
    "name": "<user defined drive name>",
    "rootFolderId": "<uuid of the drive root folder>"
}
```

##### Create Drive

###### New Drive Entity

- The user must specify a `name` of the drive which is stored within the Drive Entity's metadata JSON.
- ArDrive generates a new unique uuidv4 for the drive entity's `Drive-Id`.
- ArDrive also generates a new unique uuidv4 for the drive entity's `rootFolderId`, which will refer to the `Folder-Id` of the new folder entity that will be created.
  - This `rootFolderId` is stored within the Drive Entity's metadata JSON.
- Drive Entity Metadata transactions must have `Entity-Type: "drive"`.
- ArDrive will that the current local system time as seconds since Unix epoch for the Drive Entity's `Unix-Time`.
- The Drive Entity's `Drive-Privacy` must also be set to `public` or `private` in order for its subfolders and files to have the correct security settings.
- If the drive is private:
  - Its `Cipher` tag must be filled out with the correct encryption algorithm (currently `AES256-GCM`).
  - Its `Cipher-IV` tag must be filled out with the generated Initialization Vector for the private drive.
  - The ArFS client must derive the Drive Key and encrypt the Drive Entity's metadata JSON using the assigned `Cipher` and `Cipher-IV`.

###### New Root Folder Entity

- The `name` of the drive and folder entities must be the same.
  - This `name` is stored within the Folder Entity's metadata JSON.
- The Folder Entity's `Folder-Id` must match the `rootFolderId` previously created for the Drive Entity.
- The Folder Entity's `Drive-Id` must match the `Drive-Id` previously created for the Drive Entity.
- The Folder Entity must not include a `Parent-Folder-Id` tag.
  - This is how it is determined to be the root folder for a drive.
- Folder Entity metadata transactions must have `Entity-Type: 'folder'`.
- The client gets the user's local time for the `Unix-Time` tag, represented as seconds since Unix Epoch.
- Public folders must have the content type `Content-Type: "application/json"`.
- If the folder is private
  - Its `Cipher` tag must be filled out with the correct encryption algorithm (currently `AES256-GCM`).
  - Its `Cipher-IV` tag must be filled out with the generated Initialization Vector for the private folder.
  - Its content type must be `Content-Type: "application/octet-stream"`.
  - The ArFS client must encrypt the Drive Entity's metadata JSON using the assigned `Cipher` and `Cipher-IV`.

#### Renaming Drives

- A new root folder metadata transaction is created when a user wants to rename an existing drive.
- The new root folder metadata transaction reuses the existing drive's `Drive-Id` and `Folder-Id`, and copies all of its old metadata values, except the drive's and folder's `name` field must be updated in its data JSON to the new drive name.
  - For private drives, new ciphers are generated for this new root folder metadata transaction.
- The new root folder transaction must not have any `Parent-Folder-Id` since it is a root folder.

#### Folder Entity

A folder is a logical grouping of other folders and files. Folder entity metadata transactions without a parent folder id are considered the Drive Root Folder of their corresponding Drives. All other Folder entities must have a parent folder id. Since folders do not have underlying data, there is no Folder data transaction required.

```json
ArFS: "0.13"
Cipher?: "AES256-GCM"
Cipher-IV?: "<12 byte initialization vector as Base64>"
Content-Type: "<application/json | application/octet-stream>"
Drive-Id: "<drive uuid>"
Entity-Type: "folder"
Folder-Id: "<uuid>"
Parent-Folder-Id?: "<parent folder uuid>"
Unix-Time: "<seconds since unix epoch>"

Data JSON {
    "name": "<user defined folder name>"
}
```

##### Create Folder

Folders can be created to organize files.

- A new Folder Entity Metadata is created when a user wants to create a new folder.
- Folders can only be created in existing drives, and must have a valid `Drive-Id`.
- Folders can only be created in existing parent folders, and must have a valid `Parent-Folder-Id`.
- The new folder metadata transaction must generate a new UUIDv4 for the `Folder-Id`.
- Folder Entity Metadata transactions must have `Entity-Type: "folder"`.
- The client gets the user’s local time for the `Unix-Time` tag, represented as Seconds Since Unix Epoch.
- The user defined folder name is added to the `name` property in the folder’s metadata transaction Data JSON.
- Public folders must have the content type `Content-Type: "<application/json>"`.
- If the folder is private:
  - Its `Cipher` tag must be filled out with the respective encryption algorithm (currently `AES256-GCM`).
  - Its `Cipher-IV` tag must be filled out with the generated Initialization Vector for the private folder.
  - It must have the content type `Content-Type: "application/octet-stream"`.
  - The ArFS client must encrypt the Folder entity’s metadata JSON using the assigned `Cipher` and `Cipher-IV`.

#### Moving Folders

- A new file metadata transaction is created when a user wants to move a folder into a different folder.
- The new file metadata transaction reuses the existing folder’s `Folder-Id` and copies all of it’s old metadata values, but the file’s `Parent-Folder-Id` must be updated to the `Folder-Id` of the folder is was just moved to.
  - For private folders, new ciphers are generated for this new metadata transaction.
- Folder’s must not be allowed to be moved into a folder if another folder exists in that folder with the same folder name.

#### Renaming Folders

- A new folder metadata transaction is created when a user wants to rename an existing folder.
- The new folder metadata transaction reuses the existing folder’s `Folder-Id` and copies all of it’s old metadata values, but the folder’s `name` field in its Data JSON must be updated to the new folder name.
  - For private folders, new ciphers are generated for this new metadata transaction.
- Folders must not be allowed to be renamed to the name of another folder with that same name in that same folder.

#### File Entity

A File contains uploaded data, like a photo, document, or movie.

In the Arweave File System, a single file is broken into 2 parts - its metadata and its data.

In the Arweave File System, a single file is broken into 2 parts - its metadata and its data.

A File entity metadata transaction does not include the actual File data. Instead, the File data must be uploaded as a separate transaction, called the File Data Transaction. The File JSON metadata transaction contains a reference to the File Data Transaction ID so that it can retrieve the actual data. This separation allows for file metadata to be updated without requiring the file itself to be reuploaded. It also ensures that private files can have their JSON Metadata Transaction encrypted as well, ensuring that no one without authorization can see either the file or its metadata.

```json
ArFS: "0.13"
Cipher?: "AES256-GCM"
Cipher-IV?: "<12 byte initialization vector as Base64>"
Content-Type: "<application/json | application/octet-stream>"
Drive-Id: "<drive uuid>"
Entity-Type: "file"
File-Id: "<uuid>"
Parent-Folder-Id: "<parent folder uuid>"
Unix-Time: "<seconds since unix epoch>"

Data JSON {
    "name": "<user defined file name with extension eg. happyBirthday.jpg>",
    "size": "<computed file size - int>",
    "lastModifiedDate": "<timestamp for OS reported time of file's last modified date represented as milliseconds since unix epoch - int>",
    "dataTxId": "<transaction id of stored data>",
    "dataContentType": "<the mime type of the data associated with this file entity>",
    "pinnedDataOwner": "<the address of the original owner of the data where the file is pointing to>" // Optional
}
```

Since the version v0.13, ArFS suports Pins. Pins are files whose data may be any transaction uploaded to Arweave, that may or may not be owned by the wallet that created the pin.

When a new File Pin is created, the only created transaction is the Metadata Transaction. The `dataTxId` field will point it to any transaction in Arweave, and the optional `pinnedDataOwner` field is gonna hold the address of the wallet that owns the original copy of the data transaction.

The File Data Transaction contains limited information about the file, such as the information required to decrypt it, or the Content-Type (mime-type) needed to view in the browser.

```json
Cipher?: "AES256-GCM",
Cipher-IV?: "<12 byte initialization vector as Base64>",
Content-Type: "<file mime-type | application/octet-stream>",
 { File Data - Encrypted if private }
```

The the File Metadata Transaction contains the GQL Tags necessary to identify the file within a drive and folder.

Its data contains the JSON metadata for the file. This includes the file name, size, last modified date, data transaction id, and data content type.

```json
ArFS: "0.13",
Cipher?: "AES256-GCM",
Cipher-IV?: "<12 byte initialization vector as Base64>",
Content-Type: "<application/json | application/octet-stream>",
Drive-Id: "<drive uuid>",
Entity-Type: "file",
File-Id: "<uuid>",
Parent-Folder-Id: "<parent folder uuid>",
Unix-Time: "<seconds since unix epoch>",
 { File JSON Metadata - Encrypted if private }
```

##### Create File

- A new file metadata transaction and a separate data transaction are created when a user wants to create a new file.
- Files can only be created in existing drives, and must have a valid Drive-Id.
- Files can only be created in existing parent folders, and must have a valid Parent-Folder-Id.
- The new File Entity Data transaction must only specify the file’s mime type aka Content-Type.
- The new file metadata transaction must generate a new UUIDv4 for the File-Id.
- File metadata transactions must have Entity-Type: "file".
- The client gets the user’s local time for the Unix-Time tag, represented as Seconds Since Unix Epoch.
  - The client populates the File Entity Metadata Transaction Data JSON after creating the data transaction.
  - `name` The name of the file including extension.
  - `size` The size of the file on disk, in bytes as an integer.
  - `lastModifiedDate` The file’s last time of modification as reported by the user’s operating system, in milliseconds since Unix epoch.
  - `dataTxId` The Arweave transaction id of this File Entity’s Data Transaction.
  - `dataContentType` The mime time of this File Entity’s data must be determined by the client.
- If the File is private:
  - Its `Cipher` tag must be filled out with the respective encryption algorithm (currently `AES256-GCM`) for both the Metadata and Data transactions.
  - Its `Cipher-IV` tag must be filled out with the generated Initialization Vector for both the Metadata and Data transactions. Each one has its own unique IV.
  - It must have the content type `Content-Type: "application/octet-stream"` for both the Metadata and Data transactions.
  - The ArFS client must encrypt the File Entity’s Data and Metadata JSON using their assigned `Cipher` and `Cipher-IV`

#### Moving File

Files can be moved from one folder to another within the same Drive.

* A new file metadata transaction is created when a user wants to move a file into a different folder.
* The new file metadata transaction copies all of the file’s old metadata values, but the file’s `Parent-Folder-Id` must be updated to the `Folder-Id` of the folder is was just moved to.
    * For private files, a new `Cipher-IV` is generated for this new metadata transaction.
* File’s must not be allowed to be moved into a folder if a file exists in that folder with the same file name.

#### Renaming File

Files can be renamed from one name to another.

* A new file metadata transaction is created when a user wants to rename an existing file.
* The new file metadata transaction reuses the existing file’s `File-Id` and copies all of it’s old metadata values, but the file’s `name` field in its Data JSON must be updated to the new file name and extension.
    * For private files, a new `Cipher-IV` is generated for this new metadata transaction.
* File’s must not be allowed to be renamed to the name of another file with that same name in that same folder.

#### Updating File Version

When a user adds a new file to a folder, and there is a file in that folder with the same name, then a new file version is created.

* A new file version uses the same `File-Id` of the file with the matching name and same `Parent-Folder-Id`.  
* The file upload process is followed and new File Metadata and Data transactions are created.  
* However a new UUID is not generated and the same `File-Id` and associated metadata is used for this new version
* The new File Metadata Transaction points to the new Data transaction.
    * Since the `File-Id` remains the same, the File Keys for private files can decrypt all versions of that file.
    * For private files, new `Cipher-IV`s are generated for this new metadata and data transaction
* ArFS clients can now iterate through the state of this file, since it will have multiple versions using the same `File-Id`.



#### Snapshot Entity

ArFS applications generate the latest state of a drive by querying for all ArFS transactions made relating to a user's particular `Drive-Id`. This includes both paged queries for indexed ArFS data via GQL, as well as the ArFS JSON metadata entries for each ArFS transaction.

For small drives (less than 1000 files), a few thousand requests for very small volumes of data can be achieved relatively quickly and reliably. For larger drives, however, this results in long sync times to pull every piece of ArFS metadata when the local database cache is empty. This can also potentially trigger rate-limiting related ArWeave Gateway delays.

Once a drive state has been completely, and accurately generated, in can be rolled up into a single snapshot and uploaded as an Arweave transaction. ArFS clients can use GQL to find and retrieve this snapshot in order to rapidly reconstitute the total state of the drive, or a large portion of it. They can then query individual transactions performed after the snapshot.

This optional method offers convenience and resource efficiency when building the drive state, at the cost of paying for uploading the snapshot data. Using this method means a client will only have to iterate through a few snapshots instead of every transaction performed on the drive.

##### Snapshot Entity Tags

Snapshot entities require the following tags. These are queried by ArFS clients to find drive snapshots, organize them together with any other transactions not included within them, and build the latest state of the drive.

```json
ArFS: "0.13",
Drive-Id: "<drive uuid that this snapshot is associated with>",
Entity-Type: "snapshot",
Snapshot-Id: "<uuid of this snapshot entity>",
Content-Type: "<application/json>",
Block-Start: "<the minimum block height from which transactions were searched for in this snapshot, eg. 0>",
Block-End: "<the maximum block height from which transactions were searched for in this snapshot, eg 1007568>",
Data-Start: "<the first block in which transaction data was found in this snapshot, eg 854300",
Data-End: "<the last block in which transaction was found in this snapshot, eg 1001671",
Unix-Time: "<seconds since unix epoch>",
```

##### Snapshot Entity Data

A JSON data object must also be uploaded with every ArFS Snapshot entity. THis data contains all ArFS Drive, Folder, and File metadata changes within the associated drive, as well as any previous Snapshots. The Snapshot Data contains an array `txSnapshots`. Each item includes both the GQL and ArFS metadata details of each transaction made for the associated drive, within the snapshot's start and end period.

A `tsSnapshot` contains a `gqlNode` object which uses the same GQL tags interface returned by the Arweave Gateway. It includes all of the important `block`, `owner`, `tags`, and `bundledIn` information needed by ArFS clients. It also contains a `dataJson` object which stores the correlated Data JSON for that ArFS entity.

For private drives, the `dataJson` object contains the JSON-string-escaped encrypted text of the associated file or folder. This encrypted text uses the file's existing `Cipher` and `Cipher-IV`. This ensures clients can decrypt this information quickly using the existing ArFS privacy protocols.

```json
{
  "txSnapshots": [
    {
      "gqlNode": {
        "id": "bWCvIc3cOzwVgquD349HUVsn5Dd1_GIri8Dglok41Vg",
        "owner": {
          "address": "hlWRbyJ6WUoErm3b0wqVgd1l3LTgaQeLBhB36v2HxgY"
        },
        "bundledIn": {
          "id": "39n5evzP1Ip9MhGytuFm7F3TDaozwHuVUbS55My-MBk"
        },
        "block": {
          "height": 1062005,
          "timestamp": 1669053791
        },
        "tags": [
          {
            "name": "Content-Type",
            "value": "application/json"
          },
          {
            "name": "ArFS",
            "value": "0.11"
          },
          {
            "name": "Entity-Type",
            "value": "drive"
          },
          {
            "name": "Drive-Id",
            "value": "f27abc4b-ed6f-4108-a9f5-e545fc4ff55b"
          },
          {
            "name": "Drive-Privacy",
            "value": "public"
          },
          {
            "name": "App-Name",
            "value": "ArDrive-App"
          },
          {
            "name": "App-Platform",
            "value": "Web"
          },
          {
            "name": "App-Version",
            "value": "1.39.0"
          },
          {
            "name": "Unix-Time",
            "value": "1669053323"
          }
        ]
      },
      "dataJson": "{\"name\":\"november\",\"rootFolderId\":\"71dfc1cb-5368-4323-972a-e9dd0b1c63a0\"}"
    }
  ]
}
```

### Reading ArFS Data

Clients can perform read operations to create a timeline of entity write transactions which can then be replayed to construct the Drive state. This is done by querying an Arweave GraphQL index for the user’s respective transactions. [Arweave GraphQL Guide](https://gql-guide.vercel.app/) can provide more information on how to use Arweave GraphQL. If no GraphQL index is available, drive state can only be generated by downloading and inspecting all transactions made by the user’s wallet

This timeline of transactions should be grouped by the block number of each transaction. At every step of the timeline, the client can check if the entity was written by an authorized user. This also conveniently enables the client to surface a trusted entity version history to the user.

To determine the owner of a Drive, clients must check for who created the first Drive Entity transaction using that `Drive-Id`. Until a trusted permissions or ACL system is put in place, any transaction in a drive created by any wallet other than the one who created the first Drive Entity transaction could be considered spam.

The `Unix-Time` defined on each transaction should be reserved for tie-breaking same entity updates in the same block and should not be trusted as the source of truth for entity write ordering. This is unimportant for single owner drives but is crucial for multi-owner drives with updateable permissions (currently undefined in this spec) as a malicious user could fake the `Unix-Time` to modify the drive timeline for other users.

- Drives that have been updated many times can have a long entity timeline which can be a performance bottleneck. To avoid this, clients can cache the drive state locally and sync updates to the file system by only querying for entities in blocks higher than the last time they checked.
- Not checking for Drive Ownership could result in seeing incorrect drive state and GraphQL queries.

#### Folder/File Paths

ArweaveFS does not store folder or file paths along with entities as these paths will need to be updated whenever the parent folder name changes which can require many updates for deeply nested file systems. Instead, folder/file paths are left for the client to generate from the folder/file names.

#### Folder View Queries

Clients that want to provide users with a quick view of a single folder can simply query for an entity timeline for a particular folder by its id. Clients with multi-owner permissions will additionally have to query for the folder's parent drive entity for permission based filtering of the timeline.

### Privacy

The Arweave blockweave is inherently public. But with apps that use ArFS, like ArDrive, your private data never leaves your computer without using military grade (and [quantum resistant](https://blog.boot.dev/cryptography/is-aes-256-quantum-resistant/#:~:text=Symmetric%20encryption%2C%20or%20more%20specifically,key%20sizes%20are%20large%20enough)) encryption. This privacy layer is applied at the Drive level, and users determine whether a Drive is public or private when they first create it. Private drives must follow the ArFS privacy model.

With ArDrive specifically, every file within a Private Drive is symmetrically encrypted using [AES-256-GCM](https://iopscience.iop.org/article/10.1088/1742-6596/1019/1/012008/pdf) (for small files and metadata transactions) or [AES-256-CTR](https://xilinx.github.io/Vitis_Libraries/security/2020.1/guide_L1/internals/ctr.html) (for large files, over 100MiB). Every Private drive has a master "Drive Key" which uses a combination of the user's Arweave wallet signature, a user defined drive password, and a unique drive identifier ([uuidv4](https://en.wikipedia.org/wiki/Universally_unique_identifier)). Each file has its own "File Key" derived from the "Drive Key". This allows for single files to be shared without exposing access to the other files within the Drive.

Once a file is encrypted and stored on Arweave, it is locked forever and can only be decrypted using its file key.

**NOTE**: Usable encryption standards are not limited to AES-256-GCM or AES-256-CTR. Any Encryption method may be used so long as it is clearly indicated in the `cipher` tag.

#### Deriving Keys

Private drives have a global drive key, `D`, and multiple file keys `F`, for encryption. This enables a drive to have as many uniquely encrypted files as needed. One key is used for all versions of a single file (since new file versions use the same File-Id)

`D` is used for encrypting both Drive and Folder metadata, while `F` is used for encrypting File metadata and the actual stored data. Having these different keys, `D` and `F`, allows a user to share specific files without revealing the contents of their entire drive.

`D` is derived using HKDF-SHA256 with an [unsalted](<https://en.wikipedia.org/wiki/Salt_(cryptography)>) RSA-PSS signature of the drive's id and a user provided password.

`F` is also derived using HKDF-SHA256 with the drive key and the file's id.

Other wallets (like [ArConnect](https://www.arconnect.io/)) integrate with this Key Derivation protocol just exposing an API to collect a signature from a given Arweave Wallet in order to get the SHA-256 signature needed for the [HKDF](https://en.wikipedia.org/wiki/HKDF) to derive the Drive Key.

An example implementation, using Dart, is available [here](https://github.com/ardriveapp/ardrive-web/blob/187b3fb30808bda452123c2b18931c898df6a3fb/docs/private_drive_kdf_reference.dart), with a Typescript implementation [here](https://github.com/ardriveapp/ardrive-core-js/blob/f19da30efd30a4370be53c9b07834eae764f8535/src/utils/crypto.ts).

#### Private Drives

Drives can store either public or private data. This is indicated by the `Drive-Privacy` tag in the Drive entity metadata.

```json
Drive-Privacy: "<public | private>"
```

If a Drive entity is private, an additional tag `Drive-Auth-Mode` must also be used to indicate how the Drive Key is derived. ArDrive clients currently leverage a secure password along with the Arweave Wallet private key signature to derive the global Drive Key.

```json
Drive-Auth-Mode?: 'password'
```

On every encrypted Drive Entity, a `Cipher` tag must be specified, along with the public parameters for decrypting the data. This is done by specifying the parameter with a `Cipher-*` tag. eg. `Cipher-IV`. If the parameter is byte data, it must be encoded as Base64 in the tag.

ArDrive clients currently leverage AES256-GCM for all symmetric encryption, which requires a Cipher Initialization Vector consisting of 12 random bytes.

```json
Cipher?: "AES256-GCM"
Cipher-IV?: "<12 byte initialization vector as Base64>"
```

Additionally, all encrypted transactions must have the `Content-Type` tag `application/octet-stream` as opposed to `application/json`

Private Drive Entities and their corresponding Root Folder Entities will both use these keys and ciphers generated to symmetrically encrypt the JSON files that are included in the transaction. This ensures that only the Drive Owner (and whomever the keys have been shared with) can open the drive, discover the root folder, and continue to load the rest of the children in the drive.

#### Private Files

When a file is uploaded to a private drive, it by default also becomes private and leverages the same drive keys used for its parent drive. Each unique file in a drive will get its own set of file keys based off of that file's unique `FileId`. If a single file gets a new version, its `File-Id` will be reused, effectively leveraging the same File Key for all versions in that file's history.

These file keys can be shared by the drive's owner as needed.

Private File entities have both its metadata and data transactions encrypted using the same File Key, ensuring all facets of the data is truly private. As such, both the file's metadata and data transactions must both have a unique `Cipher-IV` and `Cipher` tag:

```json
Cipher?: "AES256-GCM"
Cipher-IV?: "<12 byte initialization vector as Base64>"
```

Just like drives, private files must have the `Content-Type` tag set as `application/octet-stream` in both its metadata and data transactions:

```json
Content-Type: "application/octet-stream"
```

### Content Types

All transaction types in ArFS leverage a specific metadata tag for the Content-Type (also known as mime-type) of the data that is included in the transaction. ArFS clients must determine what the mime-type of the data is, in order for Arweave gateways and browsers to render this content appropriately.

All public drive, folder, and file (metadata only) entity transactions all use a JSON standard, therefore they must have the following content type tag:

```json
Content-Type: '<application/json>'
```

However, a file's data transaction must have its mime-type determined. This is stored in the file's corresponding metadata transaction JSON's `dataContentType` as well as the content type tag in the data transaction itself.

```json
Content-Type: "<file's mime-type>"
```

All private drive, folder, and file entity transactions must have the following content type, since they are encrypted:

```json
Content-Type: '<application/octet-stream>'
```

### Other Tags

ArFS enabled clients should include the following tags on their transactions to identify their application

```json
App-Name: "<defined application name eg. ArDrive"
App-Version: "<defined version of the app eg. 0.5.0"
Client?: "<if the application has multiple clients, they should be specified here eg. Web" 
```



### Extending Schemas

Web app and clients can extend the ArFS Schema as needed by adding additional tags into the File and Folder MetaData Transaction JSON. This gives Developers additional flexibility to support specific application needs, without breaking the overall data model or impacting privacy.

For example a Music Sharing App could use the following expanded File Metadata for specific music files.

```json
{
    "name": "<user defined file name>",
    "size": <computed file size - int>,
    "lastModifiedDate": <timestamp for OS reported time of file's last modified date represented as milliseconds since unix epoch - int>,
    "dataTxId": "<transaction id of stored data>",
    "dataContentType": "<the mime type of the data associated with this file entity>",
    "bandName": "<the name of the band/artist>",
    "bandAlbum": "<the album of the band/artist>",
    "albumSong": "<the title of the song>"
}
```

Additionally, the above extended Metadata fields could be added directly as a transaction tag as well, in order to support GraphQL queries. 

Arweave Transaction Headers can only fit a maximum of 2048 bytes total, so this must be taken into account by clients writing custom GQL tags.

### Diagrams

Schema diagrams for public and private drives may be found [here](https://docs.ardrive.io/docs/arfs/entity-types.html#schema-diagrams).


### Additional Resources

## Resources

For more information, documentation, and community support, refer to the following resources:

- [Arweave Official Website](https://www.arweave.org/)
- [Arweave Developer Documentation](https://docs.arweave.org/)
- [Arweave Community Forums](https://community.arweave.org/)
- [ArDrive Official Documentation](https://docs.ardrive.io/)