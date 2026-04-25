# fsulib.archivematica.external-db

An Ansible role that creates and configures MySQL databases and users for [Archivematica](https://www.archivematica.org/), developed by [Florida State University Libraries](https://www.lib.fsu.edu/).

## Requirements

- Ansible 2.5 – 2.20
- The target host must be running **Ubuntu** (Jammy or Noble) or **Alpine Linux**
- The `amazon.aws` collection and `boto3`/`botocore` Python packages are required on the **controller** host when using the S3 restore feature:

```bash
ansible-galaxy collection install amazon.aws
pip install boto3 botocore
```

## Role Variables

### Connection

| Variable | Default | Description |
|---|---|---|
| `mysql_root_user` | `root` | MySQL root username |
| `mysql_root_password` | `password` | MySQL root password |
| `mysql_port` | `3306` | MySQL port |
| `mysql_host` | `localhost` | MySQL host |
| `mysql_login_host` | `%` | Host used in user grant statements |

### Databases

| Variable | Default | Description |
|---|---|---|
| `mysql_databases` | `[]` | List of databases to create (see structure below) |

Each entry in `mysql_databases` supports the following keys:

| Key | Required | Description |
|---|---|---|
| `name` | yes | Database name |
| `login_host` | yes | Host to connect to when creating the database |
| `collation` | no | Collation (default: `utf8_general_ci`) |
| `encoding` | no | Character encoding (default: `utf8`) |
| `s3_restore` | no | If present, this database will be included in S3 restore runs (see below) |

### Users

| Variable | Default | Description |
|---|---|---|
| `mysql_users` | `[]` | List of MySQL users to create (see structure below) |

Each entry in `mysql_users` supports the following keys:

| Key | Required | Description |
|---|---|---|
| `name` | yes | Username |
| `login_host` | yes | Host to connect to when creating the user |
| `pass` | no | Password (default: `pass`) |
| `priv` | no | Privilege string (default: `*.*:ALL`) |
| `append_privs` | no | Append rather than replace privileges (default: `false`) |
| `host` | no | Host the user connects from (default: `localhost`) |

### S3 Restore

| Variable | Default | Description |
|---|---|---|
| `mysql_restore_from_s3` | `false` | Set to `true` to enable the drop/recreate/reload workflow |
| `mysql_s3_aws_access_key` | `""` | AWS access key ID (omit to use an instance profile) |
| `mysql_s3_aws_secret_key` | `""` | AWS secret access key (omit to use an instance profile) |
| `mysql_s3_aws_region` | `us-east-1` | AWS region of the S3 bucket |
| `mysql_s3_staging_dir` | `/tmp/mysql_s3_restore` | Temporary directory on the target host for downloaded dumps |

To include a database in a restore run, add an `s3_restore` block to its entry in `mysql_databases`:

| Key | Required | Description |
|---|---|---|
| `s3_restore.bucket` | yes | S3 bucket name |
| `s3_restore.key` | yes | Path to the `.sql` dump file within the bucket |

Databases without an `s3_restore` block are unaffected when `mysql_restore_from_s3` is `true`.

## Dependencies

None.

## Example Playbook

### Basic usage

```yaml
- hosts: db_servers
  vars:
    mysql_databases:
      - name: archivematica
        login_host: localhost
      - name: archivematica_storage
        login_host: localhost

    mysql_users:
      - name: archivematica
        pass: "secret"
        priv: "archivematica.*:ALL"
        host: localhost
        login_host: localhost

  roles:
    - fsulib.archivematica.external-db
```

### With S3 restore

```yaml
- hosts: db_servers
  vars:
    mysql_restore_from_s3: true
    mysql_s3_aws_region: us-east-1
    # Omit key/secret to use an EC2 instance profile instead
    mysql_s3_aws_access_key: "AKIAIOSFODNN7EXAMPLE"
    mysql_s3_aws_secret_key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"

    mysql_databases:
      - name: archivematica
        login_host: localhost
        s3_restore:
          bucket: my-backup-bucket
          key: dumps/archivematica/latest.sql
      - name: archivematica_storage
        login_host: localhost
        s3_restore:
          bucket: my-backup-bucket
          key: dumps/archivematica_storage/latest.sql

    mysql_users:
      - name: archivematica
        pass: "secret"
        priv: "archivematica.*:ALL/archivematica_storage.*:ALL"
        host: localhost
        login_host: localhost

  roles:
    - fsulib.archivematica.external-db
```

## Tags

| Tag | Tasks |
|---|---|
| `dependencies` | Install system packages (`python3-mysqldb`, `mysql-client`) |
| `rootmycnf` | Write `/root/.my.cnf` with root credentials |
| `databases` | Create databases |
| `restore` | Drop, recreate, and reload databases from S3 (skipped unless `mysql_restore_from_s3: true`) |
| `users` | Create MySQL users |

## S3 Restore Workflow

When `mysql_restore_from_s3: true`, the `restore` tasks execute the following steps for each database that has an `s3_restore` block:

1. Drop the database
2. Recreate it with the configured collation and encoding
3. Download the `.sql` dump from S3 to `mysql_s3_staging_dir`
4. Import the dump into the recreated database
5. Remove the staged dump file

> **Warning:** The drop step is destructive and irreversible. Ensure your S3 dump is current before enabling this feature.

## License

BSD

## Author

Malcolm Shackelford — [Florida State University Libraries](https://www.lib.fsu.edu/)
