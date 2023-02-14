Feature | Description
--------+-------------------------
[Multi-Region Capabilities](multiregion-overview.html) | This feature gives you row-level control of how and where your data is stored to dramatically reduce read and write latencies and assist in meeting regulatory requirements in multi-region deployments.
[Follower Reads](follower-reads.html) | This feature reduces read latency in multi-region deployments by using the closest replica at the expense of reading slightly historical data.
[`BACKUP`](backup.html) | This feature creates backups of your cluster's schema and data that are consistent as of a given timestamp, stored on a service such as AWS S3, Google Cloud Storage, NFS, or HTTP storage.<br><br>[Incremental backups](take-full-and-incremental-backups.html), [backups with revision history](take-backups-with-revision-history-and-restore-from-a-point-in-time.html), [locality-aware backups](take-and-restore-locality-aware-backups.html), and [encrypted backups](take-and-restore-encrypted-backups.html) require an Enterprise license. [Full backups](take-full-and-incremental-backups.html) do not require an Enterprise license.
[Changefeeds into a Configurable Sink](create-changefeed.html) | This feature targets an allowlist of tables. For every change, it emits a record to a configurable sink, either Apache Kafka or a cloud-storage sink, for downstream processing such as reporting, caching, or full-text indexing.
[Node Map](enable-node-map.html) | This feature visualizes the geographical configuration of a cluster by plotting node localities on a world map.
[Encryption at Rest](security-reference/encryption.html#encryption-at-rest-enterprise) | Supplementing CockroachDB's encryption in flight capabilities, this feature provides transparent encryption of a node's data on the local disk. It allows encryption of all files on disk using AES in counter mode, with all key sizes allowed.
[GSSAPI with Kerberos Authentication](gssapi_authentication.html) | CockroachDB supports the Generic Security Services API (GSSAPI) with Kerberos authentication, which lets you use an external enterprise directory system that supports Kerberos, such as Active Directory.
[Single Sign-on (SSO) for DB Console](sso-db-console.html) | This feature allows you to grant access to DB Console to identities managed in an external identity provider (IdP).
[Cluster Single Sign-on (SSO](sso-sql.html) | This feature allows you to grant SQL access to {{ site.data.products.core }} clusters to identities managed in an external IdP.