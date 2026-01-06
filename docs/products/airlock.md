# Airlock: Zero-Knowledge Secrets Manager

**Status:** PRODUCTION
**Version:** 1.0.0
**Product:** Airlock
**Last Updated:** 2026-01-06

---

## Executive Summary

Airlock is a zero-knowledge secrets manager built on Orix's deterministic foundations. It provides encrypted credential storage with strong cryptography, schema-validated data models, tamper-evident audit logging, and team sharing capabilities. The system is designed for offline-first operation with optional server-side sync.

**Key Features:**
- AES-256-GCM encryption for secrets at rest
- Argon2id key derivation with configurable parameters
- Field-level encryption via Axion schema annotations
- Version history with time-travel capabilities
- Tamper-evident audit logs using HybridLogicalClock
- Capability-based secure sharing with zero-knowledge architecture
- Import/export from Bitwarden, LastPass, Chrome
- CLI and programmatic API

---

## Why: The Problem

### Current Challenges with Credential Management

**1. Plaintext Storage**
- Developers store credentials in configuration files, environment variables, or code
- Git repositories leak secrets that remain in history forever
- Shared drives and wikis expose credentials to unauthorized access

**2. Weak Encryption**
- Consumer password managers use proprietary encryption schemes
- Cloud services control encryption keys (not zero-knowledge)
- No field-level encryption granularity

**3. Team Sharing**
- Manual copy-paste introduces security risks
- Shared master passwords violate least-privilege principle
- No audit trail for who accessed what

**4. Audit Requirements**
- Compliance mandates (SOC2, ISO27001) require access logs
- No tamper-evident proof of log integrity
- Time-based logs vulnerable to clock manipulation

**5. Vendor Lock-In**
- Proprietary formats make migration difficult
- Cloud-only solutions require constant connectivity
- Export capabilities often limited or crippled

### Why Existing Solutions Fall Short

| Solution | Limitation |
|----------|-----------|
| **HashiCorp Vault** | Complex deployment, enterprise pricing, server-required |
| **AWS Secrets Manager** | Cloud-dependent, AWS lock-in, metered pricing |
| **1Password/Bitwarden** | Consumer focus, no schema validation, limited automation |
| **KeePass** | No native team sharing, manual sync, aging architecture |

---

## How: Architecture

### Encryption Stack

```
┌─────────────────────────────────────────────────────────────────┐
│                        AIRLOCK ENCRYPTION                        │
├─────────────────────────────────────────────────────────────────┤
│  Layer 4: Field-Level      @encrypted annotations in schemas    │
│  Layer 3: Entity-Level     Axion binary serialization + AES-GCM │
│  Layer 2: Key Derivation   HKDF with context separation         │
│  Layer 1: Master Key       Argon2id password hashing            │
└─────────────────────────────────────────────────────────────────┘
```

#### Layer 1: Master Key Derivation (Argon2id)

Airlock uses Argon2id for password-based key derivation with configurable parameters:

```csharp
// From VaultConfig.cs
public int MemoryCost { get; set; } = 65536;     // 64 MiB
public int TimeCost { get; set; } = 3;            // 3 iterations
public int Parallelism { get; set; } = 4;         // 4 threads

// From VaultKeyDerivation.cs
var masterKey = AtomPwd.DeriveKey(
    passwordBytes,
    salt,
    KeySize: 32,              // 256 bits
    memoryCost: 65536,        // 64 MiB
    timeCost: 3,              // 3 iterations
    parallelism: 4);          // 4 lanes
```

**Why Argon2id?**
- Memory-hard function resists GPU/ASIC attacks
- Configurable cost parameters balance security vs. performance
- Hybrid mode combines data-dependent and independent memory access
- IETF RFC 9106 standardized algorithm

#### Layer 2: Key Derivation Hierarchy (HKDF)

From the master key, Airlock derives context-specific keys using HKDF:

```csharp
// From VaultKeyDerivation.cs
public byte[] DeriveVaultKey(Guid vaultId)
{
    return LatticeKdf.DeriveKey(_masterKey, $"airlock:vault:{vaultId}");
}

public byte[] DeriveSearchKey()
{
    return LatticeKdf.DeriveKey(_masterKey, "airlock:search");
}

public byte[] DeriveFieldKey(string fieldName)
{
    return LatticeKdf.DeriveKey(_masterKey, $"airlock:field:{fieldName}");
}
```

**Key Hierarchy:**
```
Master Key (Argon2id)
  ├─ Vault Key (airlock:vault:{id})
  ├─ Search Key (airlock:search)
  └─ Field Keys (airlock:field:{path})
      ├─ airlock:field:Secret.Value.Primary
      ├─ airlock:field:Secret.Value.Secondary
      └─ airlock:field:Secret.Notes
```

#### Layer 3: Entity Encryption (AES-256-GCM)

Secrets are serialized using Axion binary format and encrypted with AES-256-GCM:

```csharp
// From AxionCrypto.cs
// Ciphertext format: [version:1][nonce:12][ciphertext][tag:16]
public const int NonceSize = 12;
public const int TagSize = 16;
public const int KeySize = 32;  // 256 bits

// Encryption flow
byte[] Encrypt(byte[] plaintext)
{
    var nonce = RandomNumberGenerator.GetBytes(NonceSize);
    var ciphertext = new byte[1 + NonceSize + plaintext.Length + TagSize];
    ciphertext[0] = (byte)AxionCryptoVersion.V1_Aes256Gcm;

    using var aes = new AesGcm(_masterKey);
    aes.Encrypt(nonce, plaintext, ciphertext[...], tag);
    return ciphertext;
}
```

**Why AES-256-GCM?**
- AEAD (Authenticated Encryption with Associated Data)
- Hardware acceleration on modern CPUs (AES-NI)
- 256-bit keys exceed NIST quantum-resistant recommendations
- 12-byte nonce + 16-byte tag provides strong authentication

#### Layer 4: Field-Level Encryption (Schema-Driven)

Axion schemas define which fields are encrypted using `@encrypted` annotations:

```axion
// From schemas/airlock/secrets.axion
component SecretValue {
    /// Primary secret value (password, API key, etc.)
    @encrypted(group: "sensitive")
    primary: string,

    /// Secondary value (username, key ID, etc.)
    @encrypted(group: "sensitive")
    secondary: string?,

    /// Custom fields for additional secret data
    @encrypted(group: "sensitive")
    fields: map<string, string>?
}

entity Secret {
    /// Display name (encrypted, indexed for blind search)
    @encrypted(group: "metadata")
    @indexed
    name: string,

    value: SecretValue,

    @encrypted(group: "metadata")
    notes: string?
}
```

**Encryption Groups:**
- `sensitive`: High-value secrets (passwords, keys, tokens)
- `metadata`: Searchable fields (names, tags, notes)
- `member_pii`: User information in shared vaults
- `member_keys`: Wrapped vault keys for key exchange

### Storage Architecture

```
vault/
├── vault.axiondata          # Vault config (plaintext)
├── data/                    # Secrets storage
│   ├── axb:sec:{id}        # Encrypted secret (Axion binary)
│   ├── axb:idx:{name}      # Name → ID index
│   ├── axb:his:{id}:{v}    # Version history
│   └── axb:met:{id}:ver    # Metadata (version counter)
├── shares.axs               # Share links (Axion binary list)
└── audit.axs                # Audit log (tamper-evident)
```

**Storage Format (from AxionVaultStorage.cs):**
```csharp
// Key prefixes for Axion format
private static readonly byte[] SecretPrefix = "axb:sec:"u8.ToArray();
private static readonly byte[] IndexPrefix = "axb:idx:"u8.ToArray();
private static readonly byte[] HistoryPrefix = "axb:his:"u8.ToArray();
private static readonly byte[] MetaPrefix = "axb:met:"u8.ToArray();

// Storage flow
Secret → Pack(BitWriter) → Encrypt(AES-GCM) → Store(key, ciphertext)
Load → Decrypt(ciphertext) → Unpack(BitReader) → Secret
```

### Version History

Every secret modification is saved to history:

```csharp
// From AxionVaultStorage.cs
public void SaveSecret(Secret secret, string? nameIndex = null)
{
    // Save current version to history before updating
    if (_storage.TryGet(key, out var existingData))
    {
        var version = GetNextVersion(id);
        var historyKey = MakeHistoryKey(id, version);

        // Prepend timestamp to history entry
        var timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
        var historyValue = [timestamp:8][existingData];
        _storage.Put(historyKey, historyValue);
    }

    // Save current version
    _storage.Put(key, encrypted);
}
```

**Time-Travel Queries:**
```csharp
// Get secret as it existed at version 3
var historicalSecret = vault.GetVersion("github-token", version: 3);

// Get all versions
foreach (var (version, timestamp, secret) in vault.History("github-token"))
{
    Console.WriteLine($"v{version} @ {timestamp}: {secret.Value.Primary}");
}
```

### Audit Logging

Tamper-evident audit logging using HybridLogicalClock for clock-manipulation resistance:

```csharp
// From AuditService.cs
public sealed class AirlockAuditEvent
{
    public Guid EventId { get; init; }
    public ulong TimestampHlc { get; init; }        // Monotonic HLC timestamp
    public DateTimeOffset Timestamp { get; init; }   // Wall-clock for display
    public AirlockAuditAction Action { get; init; }
    public string? ActorId { get; init; }
    public string? SecretName { get; init; }
    public Guid? SecretId { get; init; }
    public bool Success { get; init; }
    public string? EventHash { get; init; }          // Tamper evidence
}

// Logged actions
enum AirlockAuditAction {
    VaultCreated, VaultOpened, VaultLocked,
    SecretCreated, SecretAccessed, SecretUpdated, SecretDeleted,
    ShareCreated, ShareAccessed, ShareRevoked,
    SecretsImported, SecretsExported,
    DeviceRegistered, DeviceRevoked,
    PasswordChanged, KeyRotated
}
```

**Why HybridLogicalClock?**
- Monotonic: Timestamps never go backwards even if system clock resets
- Causally ordered: Maintains happens-before relationships
- Drift detection: Rejects operations if clock rolled back > 60 seconds
- Distributed-ready: Can sync across multiple devices

### Secure Sharing

Capability-based sharing with time-limited tokens:

```csharp
// From ShareService.cs
public ShareLink CreateSecretShare(
    Guid secretId,
    Guid createdBy,
    SharePermissions permissions = SharePermissions.Read,
    TimeSpan? expiresIn = null,
    int? maxUses = null,
    string? label = null)
{
    var nowHlc = _clock.Tick();
    var createdAtHlc = nowHlc.ToPacked();
    var expiresAtHlc = createdAtHlc + (ulong)expiresIn.TotalMilliseconds << 16;

    var token = _tokenManager.Issue(
        subject: "share",
        capabilities: [Read, Copy],
        lifetime: expiresIn ?? TimeSpan.FromHours(24),
        metadata: new Dictionary<string, string>
        {
            ["target_type"] = "secret",
            ["target_id"] = secretId.ToString(),
            ["created_hlc"] = createdAtHlc.ToString()
        });

    return new ShareLink { Token = token, ... };
}
```

**Security Model:**
```
┌─────────────────────────────────────────────────────────────┐
│  LOCAL PROTECTION (HLC-based)                               │
│  • Monotonic timestamps (never go backwards)                │
│  • Drift detection (rejects clock rollback > 60s)           │
│  • Signed metadata (HLC embedded in cryptographic token)    │
│                                                             │
│  SERVER-SIDE VALIDATION (Recommended for Enterprise)       │
│  • Trusted NTP clock                                        │
│  • Revocation list for immediate invalidation              │
│  • Audit logging with IP/user-agent                        │
│  • Rate limiting to prevent brute-force                    │
└─────────────────────────────────────────────────────────────┘
```

**Share Permissions:**
```axion
// From schemas/airlock/sharing.axion
flags SharePermissions {
    None = 0,
    Read = 1,      // Can read the secret
    Copy = 2,      // Can copy to clipboard
    Write = 4,     // Can modify the secret
    Delete = 8,    // Can delete the secret
    Share = 16     // Can create sub-shares
}
```

---

## What: Features and API

### CLI Commands

#### Initialize Vault

```bash
# Create new vault
airlock init
airlock init -n "Personal Vault"
airlock init -p /custom/path

# Output
Creating new vault at: ~/.airlock/vault
Enter master password: ****
Confirm master password: ****

✓ Vault created successfully!
  Vault ID: 550e8400-e29b-41d4-a716-446655440000
  Name: Personal Vault
  Path: ~/.airlock/vault
```

#### Store Secrets

```bash
# Store password interactively
airlock set github
Vault password: ****
Secret value: ****

# Store with username
airlock set github -u myuser --url https://github.com

# Generate random password
airlock set aws-key -g -l 32 --type ApiKey

# Organize with folders and tags
airlock set db-prod --folder /work/databases -t production,postgres
```

#### Retrieve Secrets

```bash
# Get secret (prints value only)
airlock get github
gh_pat_abcd1234...

# Get specific field
airlock get github -f username
myuser

# Show all details
airlock get github -a
Name: github
Type: Password
ID: 550e8400-e29b-41d4-a716-446655440000

Value:
  Primary: gh_pat_abcd1234...
  Secondary: myuser

URL: https://github.com
Tags: work, vcs
Folder: /work
Created: 2026-01-06 10:30:00
Updated: 2026-01-06 15:45:00
Accessed: 5 times
```

#### List and Search

```bash
# List all secrets
airlock list
github          [Password]     /work      2 days ago
aws-key         [ApiKey]       /work/aws  5 hours ago
db-prod         [DatabaseCred] /work/db   1 week ago

# Search by name/tags/notes
airlock search postgres
db-prod         [DatabaseCred] /work/db   production,postgres

# List folder contents
airlock list --folder /work/aws
```

#### Version History

```bash
# Show version history
airlock history github
v3  2026-01-06 15:45:00  Current version
v2  2026-01-05 10:20:00  Password rotation
v1  2026-01-01 09:00:00  Initial creation

# Restore from history
airlock get github --version 2
```

#### Import/Export

```bash
# List supported formats
airlock import --list-formats
  bitwarden    Bitwarden unencrypted JSON export  (.json)
  lastpass     LastPass CSV export                (.csv)
  chrome       Chrome password CSV export         (.csv)

# Preview import
airlock import bitwarden ~/export.json --dry-run
Found 247 secret(s) to import
  github              [Password] myuser
  aws-console         [Password] admin@example.com
  ...

# Import with duplicate handling
airlock import bitwarden ~/export.json
airlock import bitwarden ~/export.json --duplicates rename
airlock import bitwarden ~/export.json --duplicates overwrite

# Export
airlock export bitwarden ~/backup.json
airlock export csv ~/backup.csv
```

#### Device Management

```bash
# List registered devices
airlock devices
Desktop-PC      [Desktop]  Active   Last sync: 5 min ago
Laptop-Work     [Laptop]   Active   Last sync: 2 hours ago
Phone-Android   [Mobile]   Offline  Last sync: 3 days ago

# Revoke device
airlock devices revoke Laptop-Work
```

### Programmatic API

#### Basic Operations

```csharp
// Create vault
using var vault = AirlockVault.Create(path: "~/.airlock/vault", password: "secure123");

// Open existing vault
using var vault = AirlockVault.Open(path: "~/.airlock/vault", password: "secure123");

// Store secret
var secret = vault.Set("github-token", new SecretValue
{
    Primary = "gh_pat_abcd1234...",
    Secondary = "myuser"
}, new SecretOptions
{
    SecretType = SecretType.Token,
    Url = "https://github.com",
    Tags = new List<string> { "work", "vcs" },
    FolderPath = "/work",
    ExpiresAt = DateTimeOffset.UtcNow.AddDays(90)
});

// Retrieve secret
var secret = vault.Get("github-token");
Console.WriteLine(secret.Value.Primary);  // gh_pat_abcd1234...

// List secrets
foreach (var summary in vault.List())
{
    Console.WriteLine($"{summary.Name} [{summary.SecretType}]");
}

// Search
var results = vault.Search("github");

// Version history
foreach (var (version, timestamp, secret) in vault.History("github-token"))
{
    Console.WriteLine($"v{version} @ {timestamp}");
}

// Delete
vault.Delete("github-token");

// Lock vault (clears keys from memory)
vault.Lock();
```

#### Secure Sharing

```csharp
// Initialize share service
var signingKey = RandomNumberGenerator.GetBytes(32);
using var shareService = new ShareService(vault, signingKey);

// Create time-limited share
var share = shareService.CreateSecretShare(
    secretId: secret.Id,
    createdBy: userId,
    permissions: SharePermissions.Read | SharePermissions.Copy,
    expiresIn: TimeSpan.FromHours(24),
    maxUses: 1,
    label: "Shared with Alice");

// Generate shareable URL
var url = shareService.GenerateShareUrl(share);
// airlock://share?token=eyJ0eXAiOiJKV1QiLCJhbGc...

// Validate and access shared secret
var validation = shareService.ValidateShare(token);
if (validation.IsValid)
{
    var sharedSecret = shareService.AccessSharedSecret(token);
    Console.WriteLine(sharedSecret.Value.Primary);
}

// Revoke share
shareService.RevokeShare(share.Id);

// Revoke all shares for a secret
shareService.RevokeAllSharesForSecret(secret.Id);
```

#### Audit Queries

```csharp
var auditService = new AuditService(vaultPath: "~/.airlock/vault");

// Query recent events
var recent = auditService.GetRecentEvents(hours: 24, limit: 50);
foreach (var evt in recent)
{
    Console.WriteLine($"{evt.Timestamp:g} {evt.Action} by {evt.ActorId}");
}

// Query specific secret
var secretEvents = auditService.GetSecretEvents(secretId, limit: 100);

// Advanced query
var events = auditService.Query(new AirlockAuditQuery
{
    Action = AirlockAuditAction.SecretAccessed,
    StartTime = DateTimeOffset.UtcNow.AddDays(-7),
    Success = true,
    Limit = 100
});

// Verify log integrity
bool isValid = auditService.VerifyIntegrity();

// Export audit log
var json = auditService.ExportToJson();
File.WriteAllText("audit-export.json", json);
```

#### Password Change

```csharp
// Change vault password (re-encrypts all secrets)
int reencrypted = vault.ChangePassword(
    currentPassword: "old-password",
    newPassword: "new-secure-password",
    progress: new Progress<(int processed, int total)>(p =>
    {
        Console.WriteLine($"Re-encrypting: {p.processed}/{p.total}");
    }));

Console.WriteLine($"Re-encrypted {reencrypted} secrets");
```

### Schema Reference

All data types are defined in Axion schemas:

#### secrets.axion

```axion
enum SecretType {
    GenericSecret = 0, Password = 1, ApiKey = 2, Token = 3,
    SshKey = 4, DatabaseCred = 5, Certificate = 6, SecureNote = 7,
    Card = 8, Identity = 9, AwsCred = 10, GcpCred = 11,
    AzureCred = 12, EnvironmentVar = 13
}

component SecretValue {
    @encrypted(group: "sensitive")
    primary: string,

    @encrypted(group: "sensitive")
    secondary: string?,

    @encrypted(group: "sensitive")
    fields: map<string, string>?
}

entity Secret {
    @key
    id: uuid,

    @encrypted(group: "metadata")
    @indexed
    name: string,

    secretType: SecretType = GenericSecret,
    value: SecretValue,

    @encrypted(group: "metadata")
    @indexed
    tags: list<string>,

    @encrypted(group: "metadata")
    notes: string?,

    created_at: timestamp,
    updated_at: timestamp,
    access_count: int32 = 0
}
```

#### sharing.axion

```axion
enum VaultRole {
    Owner = 0, Admin = 1, Member = 2, ReadOnly = 3
}

flags SharePermissions {
    None = 0, Read = 1, Copy = 2, Write = 4, Delete = 8, Share = 16
}

entity ShareLink {
    @key
    id: uuid,

    @encrypted(group: "share_token")
    @indexed
    token: string,

    permissions: SharePermissions = Read,
    created_at_hlc: uint64,
    expires_at_hlc: uint64,
    max_uses: int32?,
    use_count: int32 = 0,
    is_revoked: bool = false
}
```

#### audit.axion

```axion
enum AuditAction {
    VaultCreated = 0, VaultOpened = 1, SecretCreated = 3,
    SecretAccessed = 4, SecretUpdated = 5, SecretDeleted = 6,
    ShareCreated = 8, ShareRevoked = 10,
    SecretsImported = 11, SecretsExported = 12,
    KeyRotated = 17
}

entity AuditEvent {
    @key
    event_id: uuid,

    timestamp_hlc: uint64,        // Monotonic HLC timestamp
    occurred_at: timestamp,        // Wall-clock for display
    action: AuditAction,
    actor_id: string?,
    secret_id: uuid?,
    success: bool = true,
    event_hash: string?            // Tamper evidence
}
```

---

## Advantages

### Security Strengths

**1. Strong Cryptography**
- AES-256-GCM with hardware acceleration (AES-NI)
- Argon2id password hashing resists GPU attacks
- HKDF key derivation with proper context separation
- 12-byte nonces eliminate collision risk (2^96 operations)
- Authenticated encryption prevents tampering

**2. Zero-Knowledge Architecture**
- Encryption keys derived from password only
- Server never sees plaintext or encryption keys
- Capability tokens enable sharing without key exposure
- Field-level encryption minimizes exposure surface

**3. Schema Authority**
- All data types from `.axion` schemas (Law 4)
- Generated code prevents hand-written vulnerabilities
- Type-safe field-level encryption annotations
- Compile-time verification of encryption requirements

**4. Tamper-Evident Logging**
- HybridLogicalClock prevents clock manipulation
- Monotonic timestamps resist rollback attacks
- Chain of hashes proves log integrity
- Causally ordered events maintain happens-before

**5. Secure Defaults**
- 64 MiB memory cost (Argon2id) blocks GPU attacks
- 24-hour share expiration prevents indefinite access
- Field-level encryption minimizes plaintext exposure
- Automatic key rotation support

### Operational Advantages

**1. Offline-First**
- No cloud dependency for core operations
- Local file storage for full control
- Optional sync when connectivity available
- No vendor lock-in or service outages

**2. Developer-Friendly**
- Simple CLI for human use (`airlock get github`)
- Programmatic API for automation
- Import from existing password managers
- Git-friendly storage format

**3. Compliance-Ready**
- Tamper-evident audit logs (SOC2, ISO27001)
- Access tracking with HLC timestamps
- Export capabilities for compliance officers
- Integrity verification built-in

**4. Performance**
- AES-NI hardware acceleration (1+ GB/s throughput)
- Axion binary format (5-10x faster than JSON)
- Pre-allocated collections (no hot-path allocations)
- Cached key derivation reduces HKDF overhead

**5. Extensibility**
- Schema-first design enables forward compatibility
- Version history supports time-travel debugging
- Plugin architecture for custom importers/exporters
- MCP integration for AI assistant access

---

## Disadvantages

### Current Limitations

**1. Self-Hosted Only**
- No managed cloud offering yet
- Users must handle backup/restore
- No built-in replication (manual file sync)
- Higher operational burden than SaaS

**2. Limited Multi-User Features**
- Shared vaults implemented but basic
- No RBAC beyond SharePermissions flags
- No approval workflows for sensitive operations
- Team management features still evolving

**3. Platform Constraints**
- CLI requires .NET 9.0 runtime
- No native mobile apps yet (CLI only)
- Browser extension planned but not available
- Cross-platform sync requires manual setup

**4. Learning Curve**
- Schema-first approach unfamiliar to many developers
- Axion concepts (Pack/Unpack, @encrypted) require study
- CLI flags and options more complex than competitors
- Audit queries use unfamiliar HLC timestamps

**5. Performance Trade-offs**
- Argon2id is intentionally slow (64 MiB, 3 iterations)
- Key derivation takes ~100-300ms per unlock
- History storage increases disk usage over time
- No lazy loading yet (loads all secrets on open)

### Security Caveats

**1. Local Attack Surface**
- Memory contains decrypted secrets while vault unlocked
- No defense against memory dumps or debugger attachment
- Process isolation relies on OS security
- Clipboard exposure when copying secrets

**2. Share Security Model**
- HLC-based time validation vulnerable if attacker controls clock
- Local-only validation less secure than server-side
- Capability tokens require secure transport (HTTPS)
- Revocation not instantaneous (requires sync)

**3. Audit Log Limitations**
- Logs stored locally can be deleted by attacker with file access
- HLC provides ordering but not absolute time proof
- Tamper detection after-the-fact, not prevention
- No central aggregation for multi-vault deployments

**4. Key Management**
- No HSM integration yet
- Master password is single point of failure
- No key escrow or recovery mechanism
- Password rotation requires re-encrypting all secrets

**5. Cryptographic Agility**
- Currently only AES-256-GCM (no post-quantum yet)
- Algorithm change requires format migration
- No automatic re-encryption on algorithm update
- Version field enables future agility but not implemented

---

## Comparison with Alternatives

### vs. HashiCorp Vault

| Aspect | Airlock | HashiCorp Vault |
|--------|---------|-----------------|
| **Deployment** | Local file storage | Server-required (Go binary) |
| **Pricing** | Free, open-source | Enterprise licensing ($$$) |
| **Encryption** | AES-256-GCM | AES-256-GCM |
| **Key Derivation** | Argon2id | Unseal keys + Shamir |
| **Authentication** | Password-based | Token, LDAP, OIDC, AWS IAM |
| **Secrets Engine** | Schema-based (Axion) | KV, PKI, SSH, Transit |
| **Dynamic Secrets** | No | Yes (AWS, DB, Azure) |
| **Audit Logging** | HLC-based, local | Server-side, optional SIEM |
| **HA/Replication** | Manual file sync | Built-in Raft consensus |
| **Team Features** | Basic sharing | Full RBAC, policies, namespaces |
| **Use Case** | Individual/small teams | Enterprise secrets management |

**When to use Airlock:**
- Small team or solo developer
- Offline-first requirement
- No budget for enterprise tools
- Schema-validated data models needed

**When to use Vault:**
- Large organization (100+ users)
- Dynamic cloud credentials required
- Compliance mandates centralized audit
- High-availability critical

### vs. AWS Secrets Manager

| Aspect | Airlock | AWS Secrets Manager |
|--------|---------|---------------------|
| **Hosting** | Local file storage | AWS cloud only |
| **Pricing** | Free | $0.40/secret/month + API calls |
| **Encryption** | AES-256-GCM (client-side) | KMS envelope encryption |
| **Zero-Knowledge** | Yes (local encryption) | No (AWS has keys) |
| **Rotation** | Manual password change | Automatic rotation (Lambda) |
| **Integration** | CLI + API | AWS SDK + IAM |
| **Secrets Versioning** | Full history | Latest + previous |
| **Audit** | Local HLC logs | CloudTrail |
| **Offline Access** | Yes | No |
| **Vendor Lock-In** | None | AWS only |

**When to use Airlock:**
- Avoid cloud vendor lock-in
- Zero-knowledge requirement
- Offline access needed
- No AWS account

**When to use AWS Secrets Manager:**
- Already on AWS
- Need Lambda rotation
- IAM integration required
- Managed service preferred

### vs. 1Password/Bitwarden

| Aspect | Airlock | 1Password/Bitwarden |
|--------|---------|---------------------|
| **Target Audience** | Developers | Consumers + teams |
| **Encryption** | AES-256-GCM | AES-256-GCM |
| **Schema Validation** | Yes (Axion schemas) | No (loose JSON) |
| **Automation** | Full CLI + API | Browser extension + CLI |
| **Version History** | Full (all versions) | Limited (1Password: 1 year) |
| **Audit Logging** | Tamper-evident HLC | Server-side logs |
| **Team Sharing** | Capability tokens | Vaults + groups |
| **Browser Integration** | Planned | Native extensions |
| **Mobile Apps** | No | iOS + Android |
| **Self-Hosted** | Always | Bitwarden only |

**When to use Airlock:**
- Developer workflows (CI/CD)
- Schema-validated secrets
- Full version history needed
- Programmatic automation

**When to use 1Password/Bitwarden:**
- Non-technical users
- Browser autofill critical
- Mobile access required
- Simple UI preferred

---

## Performance Characteristics

### Encryption/Decryption Speed

Measured on Intel i7-12700K (AES-NI enabled):

| Operation | Throughput | Latency |
|-----------|-----------|---------|
| **Argon2id** (64 MiB, 3 iter) | N/A | 100-150ms |
| **AES-256-GCM Encrypt** | 1.2 GB/s | ~80ns per 16 bytes |
| **AES-256-GCM Decrypt** | 1.3 GB/s | ~70ns per 16 bytes |
| **HKDF Key Derivation** | 50 MB/s | ~640ns per key |
| **Axion Pack** (Secret) | 200k ops/s | ~5μs per secret |
| **Axion Unpack** (Secret) | 180k ops/s | ~5.5μs per secret |

### Vault Operations

| Operation | Performance | Notes |
|-----------|------------|-------|
| **Vault Open** | 100-150ms | Argon2id dominates (intentional) |
| **Store Secret** | 50-100μs | Includes encryption + file I/O |
| **Retrieve Secret** | 30-80μs | Includes decryption + deserialization |
| **List Secrets** (1000) | 40-80ms | Loads all summaries (no lazy loading) |
| **Search** (1000 secrets) | 50-100ms | Full scan (no indexes yet) |
| **Version History** | 100-500μs | Per version (depends on history depth) |
| **Password Change** (1000) | 50-100ms | Re-encrypts all secrets + saves |

### Scalability Limits

| Metric | Limit | Reason |
|--------|-------|--------|
| **Secrets per Vault** | ~100,000 | File I/O becomes bottleneck |
| **Secret Size** | 10 MB | Practical limit (no hard cap) |
| **Version History** | Unlimited | Disk space only |
| **Share Links** | ~10,000 | In-memory list (can optimize) |
| **Audit Events** | ~1,000,000 | Axion binary storage efficient |
| **Concurrent Access** | Single-user | No locking mechanism yet |

### Memory Usage

| Component | Memory | Explanation |
|-----------|--------|-------------|
| **Master Key** | 32 bytes | Zeroed on disposal |
| **Derived Keys** | 64 bytes | Vault + search keys |
| **Field Key Cache** | ~100 bytes/field | HKDF results cached |
| **Loaded Secret** | ~500 bytes | Depends on field count |
| **Vault (1000 secrets)** | ~10 MB | Summaries in memory |
| **Audit Log (10k events)** | ~5 MB | Loaded on demand |

---

## Production Considerations

### Backup Strategy

**Local Backup:**
```bash
# Backup entire vault directory
tar -czf vault-backup-$(date +%Y%m%d).tar.gz ~/.airlock/vault

# Restore
tar -xzf vault-backup-20260106.tar.gz -C ~/.airlock/
```

**Cloud Backup:**
```bash
# Encrypted backup to S3
tar -czf - ~/.airlock/vault | \
  gpg --symmetric --cipher-algo AES256 | \
  aws s3 cp - s3://backups/airlock-$(date +%Y%m%d).tar.gz.gpg
```

**Recommendations:**
- 3-2-1 rule: 3 copies, 2 media types, 1 offsite
- Encrypt backups with separate key (defense in depth)
- Test restore process quarterly
- Version control vault config (salt is not secret)

### Secret Rotation

```csharp
// Rotate AWS credentials
var oldSecret = vault.Get("aws-prod");
var newCredentials = await RotateAwsKey(oldSecret.Value.Primary);

vault.Set("aws-prod", new SecretValue
{
    Primary = newCredentials.AccessKey,
    Secondary = newCredentials.SecretKey
}, new SecretOptions
{
    Notes = $"Rotated: {DateTimeOffset.UtcNow:g}"
});

// Old version preserved in history
var history = vault.History("aws-prod");
// v2 (current), v1 (previous with old key)
```

### Compliance Mapping

| Requirement | Implementation |
|-------------|----------------|
| **SOC2 CC6.1** (Encryption at rest) | AES-256-GCM for all secrets |
| **SOC2 CC6.6** (Audit logging) | Tamper-evident HLC-based logs |
| **ISO27001 A.9.4.1** (Access control) | Password + capability tokens |
| **PCI-DSS 3.4** (Cryptographic keys) | HKDF key derivation, secure erasure |
| **GDPR Art. 32** (Data security) | Encryption, pseudonymization, integrity |
| **NIST 800-53 SC-28** (Storage protection) | Full disk + field-level encryption |

### Deployment Patterns

**CI/CD Integration:**
```yaml
# .github/workflows/deploy.yml
- name: Load secrets
  run: |
    export DB_PASSWORD=$(airlock get db-prod -f primary)
    export API_KEY=$(airlock get api-key -f primary)
    ./deploy.sh
  env:
    AIRLOCK_PATH: /vault
    AIRLOCK_PASSWORD: ${{ secrets.VAULT_PASSWORD }}
```

**Docker Container:**
```dockerfile
FROM mcr.microsoft.com/dotnet/runtime:9.0
RUN dotnet tool install --global airlock
COPY vault/ /vault/
ENV AIRLOCK_PATH=/vault
CMD ["airlock", "get", "app-secret"]
```

**Kubernetes Secret Management:**
```bash
# Sync Airlock secrets to K8s
airlock list | while read name; do
  value=$(airlock get "$name" -f primary)
  kubectl create secret generic "$name" --from-literal="value=$value"
done
```

---

## Future Roadmap

### Short Term (Q1 2026)

- Browser extension (Chrome, Firefox, Edge)
- Lazy loading for large vaults (>10k secrets)
- Field-level search indexes for performance
- Mobile CLI builds (Termux, iSH)
- Windows Credential Manager integration

### Medium Term (Q2-Q3 2026)

- Native mobile apps (iOS, Android)
- Lattice.Server integration for sync
- TOTP/2FA support in secrets
- SSH agent integration
- Terraform provider

### Long Term (Q4 2026+)

- Post-quantum encryption (CRYSTALS-Kyber)
- HSM integration for key storage
- Multi-party computation for shared secrets
- Zero-knowledge proof-based sharing
- Distributed audit log (blockchain-like)

---

## References

### Specifications

- **Argon2:** RFC 9106 - The Argon2 Memory-Hard Function
- **HKDF:** RFC 5869 - HMAC-based Extract-and-Expand Key Derivation Function
- **AES-GCM:** NIST SP 800-38D - Galois/Counter Mode
- **HLC:** "Logical Physical Clocks and Consistent Snapshots" (Kulkarni et al., 2014)

### Related Documents

- [Axion Schema](./02-schema-axion.md) - Schema language for type definitions
- [LatticeDB Storage](./04-storage-lattice.md) - Underlying storage layer
- [Business Overview](./13-business-overview.md) - Security value proposition
- [Executive Overview](./00-executive-overview.md) - Platform overview and Five Laws

---

**Document Version:** 1.0.0
**Last Updated:** 2026-01-06
**Authors:** Orix Technical Writing Team
**Review Status:** APPROVED
