# Paperless-ngx with OAuth2 Protection

Docker Compose setup for [Paperless-ngx](https://docs.paperless-ngx.com/) document management system with OAuth2 authentication via oauth2-proxy, designed for GitOps deployment with Portainer.

## Features

- **Document Management** - Scan, index, and archive all your documents
- **OAuth2 Protection** - Integrated with oauth2-proxy for SSO authentication
- **Auto-Redirect** - Automatic redirect to sign-in page when not authenticated
- **PostgreSQL Database** - Reliable and performant database backend
- **Redis** - Task queue and caching
- **Fully Parameterized** - All configuration via environment variables
- **GitOps Ready** - No secrets in repository
- **Portainer Compatible** - Easy deployment via Portainer stacks
- **Encryption Ready** - Document storage paths prepared for VeraCrypt encryption

## Prerequisites

- Docker or Podman runtime
- Portainer (for GitOps deployment)
- Traefik reverse proxy with external network
- OAuth2-proxy deployed and configured (from oauth2-proxy-compose repo)
- Domain/subdomain pointing to your server

## Quick Start with Portainer

### 1. Generate Secrets

Generate a strong secret key for Paperless:

```bash
python3 -c 'import secrets; print(secrets.token_urlsafe(50))'
```

### 2. Deploy in Portainer

1. Go to **Stacks** → **Add Stack**
2. **Name**: `paperless`
3. **Build method**: Select **Repository**
4. **Repository URL**: `https://github.com/yourusername/paperlessngx-compose`
5. **Repository reference**: `main`
6. **Compose path**: `docker-compose.yml`

### 3. Configure Environment Variables

In the **Environment variables** section, copy and paste (update with your values):

```
# Domain & Traefik
PAPERLESS_HOST=paperless.yourdomain.com
TRAEFIK_CERTRESOLVER=myresolver

# Database
POSTGRES_PASSWORD=your-strong-database-password

# Security (REQUIRED!)
PAPERLESS_SECRET_KEY=your-generated-secret-from-step-1

# Admin User
PAPERLESS_ADMIN_USER=admin
PAPERLESS_ADMIN_PASSWORD=your-admin-password
PAPERLESS_ADMIN_MAIL=admin@yourdomain.com

# Timezone
PAPERLESS_TIME_ZONE=America/New_York

# User/Group (run 'id' on your system)
USERMAP_UID=1000
USERMAP_GID=1000

# Document Storage Paths (absolute paths on your server)
PAPERLESS_DATA_DIR=/path/to/paperless/data
PAPERLESS_MEDIA_DIR=/path/to/paperless/media
PAPERLESS_EXPORT_DIR=/path/to/paperless/export
PAPERLESS_CONSUME_DIR=/path/to/paperless/consume
```

**Note:** The paths should be absolute paths on your server, not relative paths.

### 4. Create Directories on Server

SSH into your server and create the storage directories:

```bash
ssh your-server.com

# Create directories
sudo mkdir -p /path/to/paperless/{data,media,export,consume}

# Set ownership (match USERMAP_UID/GID)
sudo chown -R 1000:1000 /path/to/paperless

# Set permissions
sudo chmod -R 755 /path/to/paperless
```

### 5. Deploy

Click **Deploy the stack**

### 6. Access Paperless

1. Wait 1-2 minutes for the database to initialize
2. Visit `https://paperless.yourdomain.com`
3. You'll be redirected to OAuth2-proxy sign-in
4. After authentication, you'll see the Paperless login page
5. Log in with your admin credentials (from step 3)

## Configuration Reference

### Required Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `PAPERLESS_HOST` | Hostname for Paperless | `paperless.yourdomain.com` |
| `POSTGRES_PASSWORD` | Database password | Strong random password |
| `PAPERLESS_SECRET_KEY` | Secret key for Django | Generated token from step 1 |
| `PAPERLESS_ADMIN_PASSWORD` | Initial admin password | Strong password |

### Storage Paths

| Variable | Description | Default |
|----------|-------------|---------|
| `PAPERLESS_DATA_DIR` | Application data and search index | `./data` |
| `PAPERLESS_MEDIA_DIR` | Original + archived documents | `./media` |
| `PAPERLESS_EXPORT_DIR` | Document exports | `./export` |
| `PAPERLESS_CONSUME_DIR` | Upload/import directory | `./consume` |

**Important:** Use absolute paths when deploying via Portainer.

### Optional Variables

See `.env.example` for complete list of optional configurations.

## Using Paperless

### Upload Documents

**Method 1: Web Interface**
- Click the upload button in the web interface
- Drag and drop or select files

**Method 2: Consume Directory**
- Copy documents to the consume directory
- Paperless automatically imports them
- Original files are deleted after successful import

**Method 3: Email** (requires additional configuration)
- Send documents via email
- See [Paperless documentation](https://docs.paperless-ngx.com/) for setup

### Mobile Apps

- [Paperless Mobile](https://github.com/astubenbord/paperless-mobile) (iOS/Android)
- [Paperless App](https://github.com/qcasey/paperless-app) (Android)

Configure the app with:
- **URL**: `https://paperless.yourdomain.com`
- **Username/Password**: Your admin credentials

## OAuth2 / SSO Integration

### Current Setup (Default)

The stack is protected by OAuth2-proxy forward-auth:
- Users must authenticate via OAuth2-proxy
- Then log in to Paperless with local credentials

### Advanced: Auto-Login with OAuth2

To automatically log in users after OAuth2 authentication:

1. **Enable auto-login** in Portainer environment variables:
   ```
   PAPERLESS_AUTO_LOGIN_USERNAME=auto
   PAPERLESS_DISABLE_REGULAR_LOGIN=true
   ```

2. **Create a user in Paperless**:
   - Log in as admin
   - Create a user named `auto` (or match the username from your OAuth2 provider)
   - This user will be automatically logged in after OAuth2 authentication

**Note:** This requires all users to share the same Paperless account. For per-user accounts, you'll need custom integration with OAuth2-proxy headers.

## Document Storage Encryption (Second Iteration)

The document storage paths are prepared for encryption using VeraCrypt or similar tools.

### VeraCrypt Setup (Future)

1. **Create encrypted container** on the host
2. **Mount to a directory** (e.g., `/mnt/paperless-encrypted`)
3. **Update environment variables**:
   ```
   PAPERLESS_DATA_DIR=/mnt/paperless-encrypted/data
   PAPERLESS_MEDIA_DIR=/mnt/paperless-encrypted/media
   PAPERLESS_EXPORT_DIR=/mnt/paperless-encrypted/export
   PAPERLESS_CONSUME_DIR=/mnt/paperless-encrypted/consume
   ```
4. **Auto-mount on boot** (systemd service or /etc/fstab)

**Benefits:**
- Documents encrypted at rest
- Database remains unencrypted for performance
- Easy backup of entire encrypted container

## Troubleshooting

### Check Logs

```bash
sudo podman logs paperless-webserver
sudo podman logs paperless-db
sudo podman logs paperless-redis
```

### Common Issues

1. **"Unauthorized" without redirect:**
   - Verify OAuth2-proxy is deployed and working
   - Check middleware configuration in docker-compose.yml
   - Ensure Traefik labels are correct

2. **Database connection errors:**
   - Wait 1-2 minutes for PostgreSQL to initialize
   - Check `POSTGRES_PASSWORD` matches in both services
   - Verify internal network connectivity

3. **Permission denied on document directories:**
   - Check `USERMAP_UID` and `USERMAP_GID` match directory ownership
   - Run: `sudo chown -R 1000:1000 /path/to/paperless`

4. **OCR not working:**
   - Set `PAPERLESS_OCR_LANGUAGE` to your document language
   - Add additional languages to `PAPERLESS_OCR_LANGUAGES`
   - Use `PAPERLESS_OCR_MODE=redo` or `force` if needed

5. **Documents not consumed:**
   - Check consume directory permissions
   - Verify path is absolute, not relative
   - Enable polling if using NFS: `PAPERLESS_CONSUMER_POLLING=60`

### Reset Admin Password

If you forget the admin password:

```bash
ssh your-server.com
sudo podman exec -it paperless-webserver python manage.py changepassword admin
```

## Backup

### What to Backup

1. **Database**: PostgreSQL volume (`paperless_db-data`)
2. **Documents**: `PAPERLESS_DATA_DIR` and `PAPERLESS_MEDIA_DIR`
3. **Configuration**: `.env` file (contains secrets!)

### Backup Script Example

```bash
#!/bin/bash
BACKUP_DIR="/backup/paperless/$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

# Backup database
sudo podman exec paperless-db pg_dump -U paperless paperless | gzip > "$BACKUP_DIR/db.sql.gz"

# Backup documents
sudo tar -czf "$BACKUP_DIR/documents.tar.gz" /path/to/paperless/data /path/to/paperless/media

# Backup configuration
cp .env "$BACKUP_DIR/env.backup"
```

## Updates

### Updating Paperless

In Portainer:
1. Go to **Stacks** → **paperless**
2. Click **Pull and redeploy**
3. Paperless will update to the latest version

**Note:** Check the [Paperless changelog](https://github.com/paperless-ngx/paperless-ngx/releases) for breaking changes before updating.

## Security Notes

- Never commit `.env` files with real secrets
- Use strong passwords for `POSTGRES_PASSWORD` and `PAPERLESS_ADMIN_PASSWORD`
- Keep `PAPERLESS_SECRET_KEY` secure and never share it
- Regularly update to latest Paperless version for security patches
- Consider enabling document encryption for sensitive documents
- Restrict access via OAuth2-proxy to authorized users only

## Performance Tuning

For large document collections:

1. **Increase resources** in docker-compose.yml:
   ```yaml
   deploy:
     resources:
       limits:
         memory: 2G
   ```

2. **Enable Redis persistence**:
   ```yaml
   redis:
     command: redis-server --save 60 1 --loglevel warning
   ```

3. **Optimize PostgreSQL** for your workload (see PostgreSQL documentation)

## License

MIT

## Resources

- [Paperless-ngx Documentation](https://docs.paperless-ngx.com/)
- [Paperless-ngx GitHub](https://github.com/paperless-ngx/paperless-ngx)
- [OAuth2-Proxy Documentation](https://oauth2-proxy.github.io/oauth2-proxy/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
