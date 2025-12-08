# How to Secure Your Postgres Database by Enabling TLS/SSL

## Overview

TLS/SSL encryption protects data in transit between PostgreSQL clients and servers, preventing eavesdropping and man-in-the-middle attacks.

## Prerequisites

- Docker and Docker Compose installed
- OpenSSL for certificate generation

## Step 1: Generate SSL Certificates

```bash
# Create certificates directory
mkdir -p postgres-certs
cd postgres-certs

# Generate private key
openssl genrsa -out server.key 2048

# Generate certificate signing request
openssl req -new -key server.key -out server.csr \
  -subj "/C=US/ST=State/L=City/O=Organization/CN=postgres"

# Generate self-signed certificate (valid for 365 days)
openssl x509 -req -in server.csr -signkey server.key \
  -out server.crt -days 365

# Set proper permissions
chmod 600 server.key
chmod 644 server.crt
```

## Step 2: Docker Compose Configuration

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    container_name: postgres-ssl
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: securepassword
      POSTGRES_DB: mydb
    volumes:
      - ./postgres-certs:/var/lib/postgresql/certs
      - postgres-data:/var/lib/postgresql/data
    command: >
      -c ssl=on
      -c ssl_cert_file=/var/lib/postgresql/certs/server.crt
      -c ssl_key_file=/var/lib/postgresql/certs/server.key
    ports:
      - "5432:5432"

volumes:
  postgres-data:
```

## Step 3: Start PostgreSQL with SSL

```bash
docker-compose up -d
```

## Step 4: Verify SSL is Enabled

```bash
# Connect and check SSL status
docker exec -it postgres-ssl psql -U admin -d mydb -c "SHOW ssl;"

# Expected output: ssl | on
```

## Step 5: Configure Client Connection

### Using psql

```bash
# Require SSL connection
psql "postgresql://admin:securepassword@localhost:5432/mydb?sslmode=require"

# Verify SSL connection
psql "postgresql://admin:securepassword@localhost:5432/mydb?sslmode=require" \
  -c "SELECT * FROM pg_stat_ssl WHERE pid = pg_backend_pid();"
```

### Connection String Examples

```bash
# Python (psycopg2)
postgresql://admin:securepassword@localhost:5432/mydb?sslmode=require

# Node.js (pg)
postgresql://admin:securepassword@localhost:5432/mydb?ssl=true&sslmode=require

# JDBC
jdbc:postgresql://localhost:5432/mydb?ssl=true&sslmode=require
```

## SSL Modes

| Mode | Description |
|------|-------------|
| `disable` | No SSL (not recommended) |
| `allow` | Try non-SSL, then SSL |
| `prefer` | Try SSL, then non-SSL (default) |
| `require` | Only SSL, no certificate verification |
| `verify-ca` | SSL with CA certificate verification |
| `verify-full` | SSL with full certificate verification |

## Step 6: Enforce SSL Connections

Edit `pg_hba.conf` to require SSL:

```bash
# Create custom pg_hba.conf
cat > postgres-certs/pg_hba.conf << 'EOF'
# TYPE  DATABASE        USER            ADDRESS                 METHOD
hostssl all             all             0.0.0.0/0               scram-sha-256
hostssl all             all             ::/0                    scram-sha-256
EOF
```

Update `docker-compose.yml`:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    container_name: postgres-ssl
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: securepassword
      POSTGRES_DB: mydb
    volumes:
      - ./postgres-certs:/var/lib/postgresql/certs
      - ./postgres-certs/pg_hba.conf:/var/lib/postgresql/pg_hba.conf
      - postgres-data:/var/lib/postgresql/data
    command: >
      -c ssl=on
      -c ssl_cert_file=/var/lib/postgresql/certs/server.crt
      -c ssl_key_file=/var/lib/postgresql/certs/server.key
      -c hba_file=/var/lib/postgresql/pg_hba.conf
    ports:
      - "5432:5432"

volumes:
  postgres-data:
```

## Production Considerations

### Use CA-Signed Certificates

For production, use certificates from a trusted Certificate Authority instead of self-signed certificates.

### Certificate Rotation

```bash
# Generate new certificates
openssl genrsa -out server-new.key 2048
openssl req -new -key server-new.key -out server-new.csr
openssl x509 -req -in server-new.csr -signkey server-new.key -out server-new.crt -days 365

# Replace old certificates
mv server-new.key server.key
mv server-new.crt server.crt
chmod 600 server.key

# Restart PostgreSQL
docker-compose restart
```

### Additional Security Settings

```yaml
command: >
  -c ssl=on
  -c ssl_cert_file=/var/lib/postgresql/certs/server.crt
  -c ssl_key_file=/var/lib/postgresql/certs/server.key
  -c ssl_min_protocol_version=TLSv1.2
  -c ssl_ciphers='HIGH:MEDIUM:+3DES:!aNULL'
  -c ssl_prefer_server_ciphers=on
```

## Troubleshooting

### Check SSL Configuration

```bash
docker exec -it postgres-ssl psql -U admin -d mydb -c "
SELECT name, setting FROM pg_settings 
WHERE name LIKE 'ssl%';"
```

### View Active SSL Connections

```bash
docker exec -it postgres-ssl psql -U admin -d mydb -c "
SELECT datname, usename, ssl, cipher, bits 
FROM pg_stat_ssl 
JOIN pg_stat_activity ON pg_stat_ssl.pid = pg_stat_activity.pid;"
```

### Common Issues

**Permission denied on server.key:**
```bash
chmod 600 postgres-certs/server.key
```

**Certificate verification failed:**
```bash
# Use sslmode=require instead of verify-ca/verify-full for self-signed certs
psql "postgresql://admin:securepassword@localhost:5432/mydb?sslmode=require"
```

## Testing SSL Connection

```bash
# Test with openssl
openssl s_client -connect localhost:5432 -starttls postgres

# Test connection rejection without SSL (if enforced)
psql "postgresql://admin:securepassword@localhost:5432/mydb?sslmode=disable"
# Should fail if SSL is enforced
```

## Summary

- Generated SSL certificates for PostgreSQL
- Configured Docker container with SSL enabled
- Enforced SSL-only connections via pg_hba.conf
- Verified SSL connections are working
- Implemented production-ready security settings

Your PostgreSQL database is now secured with TLS/SSL encryption.
