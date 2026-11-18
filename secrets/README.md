# Docker Secrets

This folder holds plain-text secrets mounted as files into containers. Each file is ignored by git.

## Setup

Replace the placeholder values with strong, unique passwords:

```bash
echo "your-mysql-root-password" > db_root_password
echo "your-mysql-gitea-password" > db_password
echo "your-mysql-gitea-password" > gitea_db_password
chmod 600 db_root_password db_password gitea_db_password
```

The `db_password` and `gitea_db_password` must have the same value—both are used to authenticate Gitea to MySQL.

## Security

These files are plain text. Keep them:
- Out of version control (they are git-ignored).
- Readable only by the host user (use `chmod 600`).
- Not shared or committed to any repository.

Before running the stack with real passwords:

```bash
chmod 600 secrets/*
```
