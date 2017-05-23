# Data Protection: Data at REST

Encryption is a form of data security that is required in industries such as healthcare and the payment card industry. Hadoop provides several ways to encrypt stored data or data at rest.

* **Volume or Disk encryption**: This is the lowest level of encryption, which protects data after physical theft or accidental loss of a disk volume. The entire volume is encrypted; this approach does not support finer-grained encryption of specific files or directories. In addition, volume encryption does not protect against viruses or other attacks that occur while a system is running.
* **Application level encryption**: Encryption is within an application running on top of Hadoop. This supports a higher level of granularity and prevents "rogue admin" access, but adds a layer of complexity to the application architecture.
* **HDFS data at rest encryption**: Encrypts selected files and directories stored \("at rest"\) in HDFS. This approach uses specially designated HDFS directories known as "encryption zones."

## Volume or Disk Encryption

There are many products that will provide full disk encryption. The Linux Unified Key Setup \(LUKS\) is one utility to setup disk encryption on Linux. See [https://en.wikipedia.org/wiki/Linux\_Unified\_Key\_Setup](https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup).

You can also use file system encryption such as eCryptfs. See [https://en.wikipedia.org/wiki/ECryptfs](https://en.wikipedia.org/wiki/ECryptfs).

**Note**: You will need to consider how to deal with providing password on boot, adding disks, resizing disks, etc. when using disk encryption

## Application Level Encryption

You can decide to encrypt and decrypt data at the application level. While this might be more onerous, it is more precise as the application knows which data needs to be encrypted or not.

## HDFS Data Encryption

HDFS data at rest encryption implements end-to-end encryption of data read from and written to HDFS. End-to-end encryption means that data is encrypted and decrypted only by the client. HDFS does not have access to unencrypted data or keys.

HDFS encryption involves several elements:

* **Encryption key**: A new level of permission-based access protection, in addition to standard HDFS permissions.
* **HDFS encryption zone**: A special HDFS directory within which all data is encrypted upon write, and decrypted upon read
  * Each encryption zone is associated with an encryption key that is specified when the zone is created. 
  * Each file within an encryption zone has a unique encryption key, called the "data encryption key" \(DEK\). 
  * HDFS does not have access to DEKs. HDFS DataNodes only see a stream of encrypted bytes. HDFS stores "encrypted data encryption keys" \(EDEKs\) as part of the file's metadata on the NameNode.
  * Clients decrypt an EDEK and use the associated DEK to encrypt and decrypt data during write and read operations.

### Create an Encryption Key

Create a "master" encryption key for the new encryption zone. Each key will be specific to an encryption zone.

Key size can be 128 or 256 bits.

**Recommendation**: Create a new superuser for key management. In the following examples, superuser encr creates the key. This separates the data access role from the encryption role, strengthening security.

Example:

`# su - encr`

`# hadoop key create key1`

To verify creation of the key, list the metadata associated with the current user:

`# hadoop key list -metadata`

**Warning**: Do not delete an encryption key while it is in use for an encryption zone. This will result in loss of access to data in that zone.

### Create an Encryption Zone

Each encryption zone must be defined using an empty directory and an existing encryption key. An encryption zone cannot be created on top of a directory that already contains data.

**Recommendation**: use one unique key for each encryption zone.

Follow these steps:

1. As HDFS administrator, create a new empty directory. For example:  
   `# hdfs dfs -mkdir /zone_encr`

2. Using the encryption key, make the directory an encryption zone. For example:  
   `# hdfs crypto -createZone -keyName key1 -path /zone_encr`
   When finished, the NameNode will recognize the folder as an HDFS encryption zone.

3. To verify creation of the new encryption zone, run the crypto -listZones command as an HDFS administrator.     You should see the encryption zone and its key. For example:
   `$ hdfs crypto -listZones /zone-encr key1`

**Note**: The following property \(in the hdfs-default.xml file\) causes listZone requests to be batched. This improves NameNode performance. The property specifies the maximum number of zones that will be returned in a batch.

`dfs.namenode.list.encryption.zones.num.responses`

The default is 100.

To remove an encryption zone, delete the root directory of the zone. For example:

`hdfs dfs -rm -R /zone_encr`

### Creating an HDFS Admin User

To capitalize on the capabilities of HDFS data at rest encryption, you will need two separate types of HDFS administrative accounts:

1. **HDFS administrative user**: an account in the hdfs supergroup that is used to manage encryption keys and encryption zones. Examples in this chapter use an administrative user account named encr.
2. **HDFS service user**: the system-level account traditionally associated with HDFS. By default this is user hdfs in HDP. This account owns the HDFS DataNode and NameNode processes.


For more details regarding HDFS transparent encryption see http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/TransparentEncryption.html
