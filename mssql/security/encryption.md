# SQL Server Encryption --- TDE, Always Encrypted, and the Key Hierarchy

## Overview

SQL Server provides multiple encryption layers to protect data at rest, in transit, and in use. **Transparent Data Encryption (TDE)** encrypts the entire database at the storage level without application changes. **Always Encrypted** protects sensitive columns so that even database administrators cannot read plaintext values. **Cell-level encryption** offers granular, application-controlled encryption for individual columns using symmetric or asymmetric keys.

The encryption hierarchy in SQL Server follows a strict chain of trust: the Service Master Key protects the Database Master Key, which protects certificates and asymmetric keys, which in turn protect symmetric keys used for data encryption. Understanding this hierarchy is critical for key management, backup/restore operations, and disaster recovery planning.

This skill covers TDE setup and rotation, Always Encrypted with deterministic and randomized encryption, backup encryption, SSL/TLS configuration, the full key hierarchy, cell-level encryption functions, and Azure Key Vault integration.

## Key Concepts

- **Service Master Key (SMK)**: Instance-level key auto-generated at installation; protects all Database Master Keys. Encrypted by DPAPI.
- **Database Master Key (DMK)**: Database-level key that protects certificates, asymmetric keys, and symmetric keys within that database.
- **TDE (Transparent Data Encryption)**: Encrypts data files (MDF), log files (LDF), and backups using a Database Encryption Key (DEK) protected by a certificate or asymmetric key.
- **Always Encrypted**: Client-side encryption where the database engine never sees plaintext. Uses Column Master Keys (CMK) and Column Encryption Keys (CEK).
- **Enclave**: A secure memory region (VBS or SGX) where SQL Server can perform operations on Always Encrypted data in plaintext (Always Encrypted with secure enclaves).
- **Cell-Level Encryption**: Application-managed encryption using ENCRYPTBYKEY / DECRYPTBYKEY functions for individual column values.

## Encryption Hierarchy

```sql
-- The SQL Server encryption hierarchy:
-- DPAPI (Windows) / EKMS
--   └── Service Master Key (instance-level, auto-created)
--         └── Database Master Key (per-database)
--               ├── Certificate
--               │     └── Symmetric Key (for data encryption)
--               ├── Asymmetric Key
--               │     └── Symmetric Key (for data encryption)
--               └── Symmetric Key (direct)

-- View the Service Master Key
SELECT * FROM sys.symmetric_keys WHERE name = '##MS_ServiceMasterKey##';

-- Back up the Service Master Key
BACKUP SERVICE MASTER KEY TO FILE = 'C:\Keys\SMK_backup.key'
    ENCRYPTION BY PASSWORD = 'SMK!B@ckup#2024';

-- Create a Database Master Key
USE AppDB;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'DMK!Str0ng#P@ss';

-- Back up the Database Master Key
BACKUP MASTER KEY TO FILE = 'C:\Keys\AppDB_DMK.key'
    ENCRYPTION BY PASSWORD = 'DMK!B@ckup#2024';
```

## Transparent Data Encryption (TDE)

```sql
-- Step 1: Create a master key in the master database
USE master;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'M@ster!K3y#2024';

-- Step 2: Create a certificate protected by the master key
CREATE CERTIFICATE TDE_Certificate
    WITH SUBJECT = 'TDE Certificate for AppDB',
    EXPIRY_DATE = '2030-12-31';

-- Step 3: Back up the certificate (critical for restore/DR)
BACKUP CERTIFICATE TDE_Certificate
    TO FILE = 'C:\Keys\TDE_Cert.cer'
    WITH PRIVATE KEY (
        FILE = 'C:\Keys\TDE_Cert.pvk',
        ENCRYPTION BY PASSWORD = 'C3rt!B@ckup#2024'
    );

-- Step 4: Create the Database Encryption Key in the target database
USE AppDB;
CREATE DATABASE ENCRYPTION KEY
    WITH ALGORITHM = AES_256
    ENCRYPTION BY SERVER CERTIFICATE TDE_Certificate;

-- Step 5: Enable TDE
ALTER DATABASE AppDB SET ENCRYPTION ON;

-- Monitor TDE status
SELECT
    db.name AS DatabaseName,
    dek.encryption_state,
    CASE dek.encryption_state
        WHEN 0 THEN 'No encryption key'
        WHEN 1 THEN 'Unencrypted'
        WHEN 2 THEN 'Encryption in progress'
        WHEN 3 THEN 'Encrypted'
        WHEN 4 THEN 'Key change in progress'
        WHEN 5 THEN 'Decryption in progress'
        WHEN 6 THEN 'Protection change in progress'
    END AS EncryptionStateDesc,
    dek.key_algorithm,
    dek.key_length,
    dek.percent_complete,
    dek.encryption_state_desc
FROM sys.dm_database_encryption_keys dek
JOIN sys.databases db ON dek.database_id = db.database_id;

-- Rotate TDE certificate
USE master;
CREATE CERTIFICATE TDE_Certificate_v2
    WITH SUBJECT = 'TDE Certificate v2 for AppDB';

USE AppDB;
ALTER DATABASE ENCRYPTION KEY
    ENCRYPTION BY SERVER CERTIFICATE TDE_Certificate_v2;

-- Back up new certificate immediately
USE master;
BACKUP CERTIFICATE TDE_Certificate_v2
    TO FILE = 'C:\Keys\TDE_Cert_v2.cer'
    WITH PRIVATE KEY (
        FILE = 'C:\Keys\TDE_Cert_v2.pvk',
        ENCRYPTION BY PASSWORD = 'C3rt!V2#B@ckup#2024'
    );
```

## Always Encrypted

```sql
-- Always Encrypted uses two key types:
-- Column Master Key (CMK): stored externally (cert store, Azure Key Vault)
-- Column Encryption Key (CEK): stored in database, encrypted by CMK

-- Step 1: Create a Column Master Key definition (points to external key)
CREATE COLUMN MASTER KEY [CMK_Auto1]
WITH (
    KEY_STORE_PROVIDER_NAME = N'MSSQL_CERTIFICATE_STORE',
    KEY_PATH = N'CurrentUser/My/A1B2C3D4E5F6...'
);

-- Step 2: Create a Column Encryption Key
CREATE COLUMN ENCRYPTION KEY [CEK_Auto1]
WITH VALUES (
    COLUMN_MASTER_KEY = [CMK_Auto1],
    ALGORITHM = 'RSA_OAEP',
    ENCRYPTED_VALUE = 0x01700000016...
);

-- Step 3: Create table with encrypted columns
CREATE TABLE HR.Employees (
    EmployeeID INT PRIMARY KEY,
    FirstName NVARCHAR(50),
    LastName NVARCHAR(50),
    SSN CHAR(11) COLLATE Latin1_General_BIN2
        ENCRYPTED WITH (
            COLUMN_ENCRYPTION_KEY = [CEK_Auto1],
            ENCRYPTION_TYPE = DETERMINISTIC,
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
        ),
    Salary MONEY
        ENCRYPTED WITH (
            COLUMN_ENCRYPTION_KEY = [CEK_Auto1],
            ENCRYPTION_TYPE = RANDOMIZED,
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
        )
);

-- Deterministic: same plaintext always produces same ciphertext
--   Supports equality comparisons, joins, GROUP BY, indexing
-- Randomized: same plaintext produces different ciphertext each time
--   More secure but no equality operations allowed

-- Query encrypted columns (requires client driver with Column Encryption enabled)
-- Connection string: Column Encryption Setting=Enabled
-- The driver handles encryption/decryption transparently

-- Rotate Column Master Key
-- Step 1: Create new CMK in key store
CREATE COLUMN MASTER KEY [CMK_Auto2]
WITH (
    KEY_STORE_PROVIDER_NAME = N'MSSQL_CERTIFICATE_STORE',
    KEY_PATH = N'CurrentUser/My/B2C3D4E5F6A1...'
);

-- Step 2: Re-encrypt the CEK with the new CMK
ALTER COLUMN ENCRYPTION KEY [CEK_Auto1]
ADD VALUE (
    COLUMN_MASTER_KEY = [CMK_Auto2],
    ALGORITHM = 'RSA_OAEP',
    ENCRYPTED_VALUE = 0x01700000017...
);

-- Step 3: Remove old CMK value from CEK
ALTER COLUMN ENCRYPTION KEY [CEK_Auto1]
DROP VALUE (COLUMN_MASTER_KEY = [CMK_Auto1]);

-- Step 4: Drop old CMK metadata
DROP COLUMN MASTER KEY [CMK_Auto1];
```

## Always Encrypted with Secure Enclaves

```sql
-- Enable enclave support (SQL Server 2019+)
-- Requires VBS (Virtualization Based Security) or SGX attestation

-- Configure enclave type (instance-level)
EXEC sp_configure 'column encryption enclave type', 1;  -- 1 = VBS
RECONFIGURE;

-- Create an enclave-enabled CMK
CREATE COLUMN MASTER KEY [CMK_Enclave]
WITH (
    KEY_STORE_PROVIDER_NAME = N'MSSQL_CERTIFICATE_STORE',
    KEY_PATH = N'CurrentUser/My/C3D4E5F6A1B2...',
    ENCLAVE_COMPUTATIONS (SIGNATURE = 0x...)
);

-- With enclaves, randomized columns support:
--   Pattern matching (LIKE)
--   Range comparisons (<, >, BETWEEN)
--   Sorting (ORDER BY)
--   In-place encryption (ALTER TABLE ... ALTER COLUMN)

-- In-place encryption of existing columns (enclave-enabled)
ALTER TABLE HR.Employees
    ALTER COLUMN SSN CHAR(11) COLLATE Latin1_General_BIN2
    ENCRYPTED WITH (
        COLUMN_ENCRYPTION_KEY = [CEK_Auto1],
        ENCRYPTION_TYPE = DETERMINISTIC,
        ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
    );
```

## Backup Encryption

```sql
-- Create a certificate for backup encryption
USE master;
CREATE CERTIFICATE BackupEncryptCert
    WITH SUBJECT = 'Backup Encryption Certificate';

-- Create an encrypted backup
BACKUP DATABASE AppDB
    TO DISK = 'C:\Backups\AppDB_encrypted.bak'
    WITH ENCRYPTION (
        ALGORITHM = AES_256,
        SERVER CERTIFICATE = BackupEncryptCert
    ),
    COMPRESSION,
    STATS = 10;

-- Encrypted log backup
BACKUP LOG AppDB
    TO DISK = 'C:\Backups\AppDB_log_encrypted.trn'
    WITH ENCRYPTION (
        ALGORITHM = AES_256,
        SERVER CERTIFICATE = BackupEncryptCert
    );

-- Back up the encryption certificate (required for restore on another server)
BACKUP CERTIFICATE BackupEncryptCert
    TO FILE = 'C:\Keys\BackupEncryptCert.cer'
    WITH PRIVATE KEY (
        FILE = 'C:\Keys\BackupEncryptCert.pvk',
        ENCRYPTION BY PASSWORD = 'B@ckup!C3rt#2024'
    );

-- Restore encrypted backup on a different server
-- Step 1: Restore the certificate
USE master;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'N3w!M@ster#K3y';

CREATE CERTIFICATE BackupEncryptCert
    FROM FILE = 'C:\Keys\BackupEncryptCert.cer'
    WITH PRIVATE KEY (
        FILE = 'C:\Keys\BackupEncryptCert.pvk',
        DECRYPTION BY PASSWORD = 'B@ckup!C3rt#2024'
    );

-- Step 2: Restore the database
RESTORE DATABASE AppDB
    FROM DISK = 'C:\Backups\AppDB_encrypted.bak'
    WITH REPLACE;
```

## SSL/TLS for Connections

```sql
-- Force encryption at the instance level (requires certificate)
-- Via SQL Server Configuration Manager or registry:
EXEC xp_instance_regwrite
    N'HKEY_LOCAL_MACHINE',
    N'Software\Microsoft\MSSQLServer\MSSQLServer\SuperSocketNetLib',
    N'ForceEncryption', REG_DWORD, 1;

-- Check if connections are encrypted
SELECT
    session_id,
    encrypt_option,
    auth_scheme,
    net_transport,
    client_net_address
FROM sys.dm_exec_connections
WHERE session_id = @@SPID;

-- View TLS protocol version for current connection
SELECT
    session_id,
    encrypt_option,
    protocol_type,
    net_transport
FROM sys.dm_exec_connections
WHERE session_id = @@SPID;

-- Query all connection encryption status
SELECT
    encrypt_option,
    COUNT(*) AS ConnectionCount
FROM sys.dm_exec_connections
GROUP BY encrypt_option;
```

## Cell-Level Encryption

```sql
-- Create a symmetric key for cell-level encryption
USE AppDB;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'DMK!P@ss#2024';

CREATE CERTIFICATE DataEncryptCert
    WITH SUBJECT = 'Data Encryption Certificate';

CREATE SYMMETRIC KEY DataEncryptKey
    WITH ALGORITHM = AES_256
    ENCRYPTION BY CERTIFICATE DataEncryptCert;

-- Encrypt data
OPEN SYMMETRIC KEY DataEncryptKey
    DECRYPTION BY CERTIFICATE DataEncryptCert;

INSERT INTO HR.EmployeeSensitive (EmployeeID, EncryptedSSN)
VALUES (1, ENCRYPTBYKEY(KEY_GUID('DataEncryptKey'), '123-45-6789'));

CLOSE SYMMETRIC KEY DataEncryptKey;

-- Decrypt data
OPEN SYMMETRIC KEY DataEncryptKey
    DECRYPTION BY CERTIFICATE DataEncryptCert;

SELECT
    EmployeeID,
    CONVERT(VARCHAR(11), DECRYPTBYKEY(EncryptedSSN)) AS SSN
FROM HR.EmployeeSensitive;

CLOSE SYMMETRIC KEY DataEncryptKey;

-- Encrypt with authenticator (prevents column swapping attacks)
OPEN SYMMETRIC KEY DataEncryptKey
    DECRYPTION BY CERTIFICATE DataEncryptCert;

UPDATE HR.EmployeeSensitive
SET EncryptedSSN = ENCRYPTBYKEY(
    KEY_GUID('DataEncryptKey'),
    '123-45-6789',
    1,                          -- use authenticator
    CAST(EmployeeID AS VARCHAR) -- authenticator value
)
WHERE EmployeeID = 1;

-- Decrypt with authenticator
SELECT
    EmployeeID,
    CONVERT(VARCHAR(11), DECRYPTBYKEY(
        EncryptedSSN,
        1,
        CAST(EmployeeID AS VARCHAR)
    )) AS SSN
FROM HR.EmployeeSensitive;

CLOSE SYMMETRIC KEY DataEncryptKey;
```

## Azure Key Vault Integration

```sql
-- Use Azure Key Vault as the CMK store for Always Encrypted
CREATE COLUMN MASTER KEY [CMK_AKV]
WITH (
    KEY_STORE_PROVIDER_NAME = N'AZURE_KEY_VAULT',
    KEY_PATH = N'https://myvault.vault.azure.net/keys/MyAlwaysEncryptedKey/abc123...'
);

-- Use EKM provider for TDE with Azure Key Vault
-- Step 1: Install SQL Server Connector for Azure Key Vault
-- Step 2: Create credential mapped to Azure Key Vault
CREATE CREDENTIAL AKV_Credential
    WITH IDENTITY = 'ContosoKeyVault',
    SECRET = 'AAD_CLIENT_ID+AAD_CLIENT_SECRET'
    FOR CRYPTOGRAPHIC PROVIDER AzureKeyVault_EKM;

-- Step 3: Create login credential mapping
ALTER LOGIN [SQLServiceLogin]
    ADD CREDENTIAL AKV_Credential;

-- Step 4: Create asymmetric key from Azure Key Vault
CREATE ASYMMETRIC KEY TDE_AKV_Key
    FROM PROVIDER AzureKeyVault_EKM
    WITH PROVIDER_KEY_NAME = 'SQLServerTDEKey',
    CREATION_DISPOSITION = OPEN_EXISTING;

-- Step 5: Create login from asymmetric key
CREATE LOGIN TDE_AKV_Login FROM ASYMMETRIC KEY TDE_AKV_Key;
GRANT CONTROL SERVER TO TDE_AKV_Login;

-- Step 6: Use AKV key for TDE
USE AppDB;
CREATE DATABASE ENCRYPTION KEY
    WITH ALGORITHM = AES_256
    ENCRYPTION BY SERVER ASYMMETRIC KEY TDE_AKV_Key;

ALTER DATABASE AppDB SET ENCRYPTION ON;
```

## Best Practices

- Always back up TDE certificates and private keys immediately after creation; without them, encrypted databases and backups are unrecoverable.
- Use Always Encrypted with randomized encryption for maximum security; use deterministic only when equality operations are required.
- Enable TDE on tempdb by encrypting any user database (tempdb auto-enables); plan for the 3-5% CPU overhead.
- Store Column Master Keys in Azure Key Vault or a Hardware Security Module, not in the local certificate store for production.
- Use backup encryption in addition to TDE to protect backups stored off-server or in cloud storage.
- Enforce TLS 1.2 or higher for all connections; disable older protocols (SSL 3.0, TLS 1.0, TLS 1.1).
- Rotate encryption keys on a regular schedule: TDE certificates annually, symmetric keys based on data sensitivity.
- Use authenticators with ENCRYPTBYKEY to prevent ciphertext value swapping between rows.
- Test backup restore procedures with encrypted backups regularly to confirm certificate availability.
- Document the encryption hierarchy and key dependencies for disaster recovery.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Not backing up the TDE certificate and private key | Database cannot be restored on another server; data loss | Back up immediately after creation, store securely off-server |
| Using cell-level encryption without authenticators | Attackers can swap encrypted values between rows | Always use the authenticator parameter with ENCRYPTBYKEY |
| Forgetting that TDE encrypts tempdb too | Performance impact on all databases, not just encrypted ones | Plan for shared tempdb encryption overhead at the instance level |
| Storing CMK in local cert store for production Always Encrypted | Key exposed if server is compromised | Use Azure Key Vault or HSM for production Column Master Keys |
| Not rotating encryption keys on a schedule | Longer key lifetimes increase exposure risk | Establish rotation procedures and automate with SQL Agent jobs |
| Enabling TDE on a database in an AG without provisioning certs on secondaries | Failover fails, secondary cannot read encrypted data | Deploy TDE certificate to all AG replicas before enabling TDE |

## SQL Server Version Notes

- **SQL Server 2016**: Introduced Always Encrypted (deterministic and randomized encryption types), TDE available in Standard Edition (previously Enterprise only). Backup encryption added.
- **SQL Server 2019**: Always Encrypted with secure enclaves (VBS and SGX), enabling rich queries on encrypted data including LIKE, range comparisons, and in-place encryption. Improved TDE with hardware acceleration. Suspend and resume TDE scan.
- **SQL Server 2022**: TDE with auto-rotation of server-level certificates. Always Encrypted enclaves improvements with VBS attestation on all Windows editions. Integration with Azure Key Vault for automatic key management. TLS 1.3 support. Ledger database with built-in cryptographic verification.

## Sources

- [Transparent Data Encryption (TDE)](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/transparent-data-encryption)
- [Always Encrypted](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/always-encrypted-database-engine)
- [Always Encrypted with Secure Enclaves](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/always-encrypted-enclaves)
- [Encryption Hierarchy](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/encryption-hierarchy)
- [Backup Encryption](https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/backup-encryption)
- [Extensible Key Management (Azure Key Vault)](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/extensible-key-management-using-azure-key-vault-sql-server)
