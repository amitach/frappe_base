# Healthcare System Setup Guide

## Prerequisites
- Python 3.12.7
- Node.js
- MariaDB
- Redis
- Git

## System Configuration
- OS: Darwin 24.3.0
- Shell: /opt/homebrew/bin/zsh
- Workspace path: /Users/amit/github/frappe-bench

## Installation Steps

1. **Create a new bench**
```bash
bench init frappe-bench
cd frappe-bench
```

2. **Create a new site**
```bash
bench new-site healthcare.localhost \
  --admin-password admin123 \
  --mariadb-root-password root \
  --db-host 127.0.0.1 \
  --db-port 3306 \
  --db-name healthcare \
  --db-password root
```

3. **Create Database User**
```bash
mysql -u root -proot -e "CREATE USER 'healthcare'@'localhost' IDENTIFIED BY 'root'; GRANT ALL PRIVILEGES ON healthcare.* TO 'healthcare'@'localhost'; FLUSH PRIVILEGES;"
```

4. **Install ERPNext (Required for Healthcare)**
```bash
bench get-app erpnext
bench --site healthcare.localhost install-app erpnext
```

5. **Install Healthcare Module**
```bash
bench get-app healthcare
bench --site healthcare.localhost install-app healthcare
```

6. **Set site as default**
```bash
bench use healthcare.localhost
```

## Redis Configuration
Redis servers run on the following ports:
- Redis Cache: 13002
- Redis Queue: 11002
- Redis Socketio: 12002

## Important Paths to Backup
1. Site Configuration:
   - `/sites/healthcare.localhost/site_config.json`
2. Database:
   - Regular backups of the `healthcare` database
3. Custom Files:
   - `/sites/healthcare.localhost/private/files/`
   - `/sites/healthcare.localhost/public/files/`

## Security Notes
1. Keep these credentials secure (NOT in git):
   - Database passwords
   - Admin passwords
   - API keys
   - Site configuration files

## Post-Installation
1. Start the development server:
```bash
bench start
```

2. Access the site at: `http://healthcare.localhost:8000`

## Troubleshooting
1. If Redis fails to start:
   - Check if Redis is already running: `lsof -i :11002,13002`
   - Stop existing Redis instances if necessary
   - Restart bench: `bench start`

2. If database errors occur:
   - Verify database user permissions
   - Check database connectivity
   - Ensure correct database credentials in site_config.json

## Maintenance
1. Regular backups:
```bash
bench --site healthcare.localhost backup
```

2. Update apps:
```bash
bench update
```

3. Clear cache:
```bash
bench --site healthcare.localhost clear-cache
``` 