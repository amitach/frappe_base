# Frappe Bench Setup Documentation

## Configuration Files

### Supervisor Configuration
- Location: `/Users/amit/supervisor/supervisord.conf`
- Key settings:
  - Unix HTTP server socket: `/tmp/supervisor.sock` (chmod: 0700)
  - Log file: `~/supervisor/logs/supervisord.log`
  - PID file: `/tmp/supervisord.pid`
  - Log rotation: 50MB max size, 10 backups
  - Additional configs: `~/supervisor/conf.d/*.conf`

### Redis Queue Configuration
- Location: `/Users/amit/github/frappe-bench/config/redis_queue.conf`
- Settings:
  - Database file: `redis_queue.rdb`
  - Directory: `/Users/amit/github/frappe-bench/config/pids`
  - Bind address: `127.0.0.1`
  - Port: `11000`
  - Save intervals:
    - 900 seconds if at least 1 key changed
    - 300 seconds if at least 10 keys changed
    - 60 seconds if at least 10000 keys changed

## Database Setup
- Database user: `healthcare`
- Host: `127.0.0.1`
- Port: `3306`
- Database name: `healthcare`
- Permissions: ALL PRIVILEGES on `healthcare.*`

## Site Setup
- Site name: `healthcare.localhost`
- Admin password: `admin123`
- Database configuration:
  - Root password: `root`
  - Database host: `127.0.0.1`
  - Database port: `3306`
  - Database name: `healthcare`
  - Database password: `root`

## Environment
- Python version: 3.12.7
- Virtual environment: `.venv`
- OS: Darwin 24.3.0
- Shell: `/opt/homebrew/bin/zsh`
- Workspace path: `/Users/amit/github/frappe-bench`

## Notes
- When creating a new site, wait a few seconds for MariaDB to start completely
- If recreating an existing site, use the `--force` flag with `bench new-site`

# Frappe Bench Setup

This repository contains a Frappe bench setup for development. Below are the setup instructions and important notes.

## Prerequisites

- Python 3.10+
- Node.js 18+
- MariaDB 10.6+ (running in Docker)
- Redis (installed via bench)
- Git
- Supervisor (for process management)

## Initial Setup

1. Clone this repository:
```bash
git clone --recursive https://github.com/yourusername/frappe-bench.git
cd frappe-bench
```

2. Set up MariaDB in Docker:
```bash
docker run -d \
  --name mariadb \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=root \
  mariadb:10.6
```

3. Install Python dependencies:
```bash
python -m venv env
source env/bin/activate
cd apps/frappe
pip install -e .
cd ../..
bench setup requirements
```

4. Create a new site:
```bash
# Create database
mysql -h127.0.0.1 -P3306 -uroot -proot -e "CREATE DATABASE healthcare CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# Create database user
mysql -h127.0.0.1 -P3306 -uroot -proot -e "CREATE USER IF NOT EXISTS 'healthcare'@'localhost' IDENTIFIED BY 'root'; GRANT ALL PRIVILEGES ON healthcare.* TO 'healthcare'@'localhost'; FLUSH PRIVILEGES;"

# Create new site
bench new-site healthcare.localhost \
  --admin-password admin123 \
  --mariadb-root-password root \
  --db-host 127.0.0.1 \
  --db-port 3306 \
  --db-name healthcare \
  --db-password root \
  --db-user healthcare
```

5. Set up Supervisor:
```bash
# Install supervisor
brew install supervisor  # On macOS
# OR
sudo apt-get install supervisor  # On Ubuntu/Debian

# Create supervisor directories
mkdir -p ~/supervisor/conf.d
mkdir -p ~/supervisor/logs

# Create main supervisor config
cat > ~/supervisor/supervisord.conf << EOL
[unix_http_server]
file=/tmp/supervisor.sock

[supervisord]
logfile=~/supervisor/logs/supervisord.log
logfile_maxbytes=50MB
logfile_backups=10
loglevel=info
pidfile=/tmp/supervisord.pid
nodaemon=false
minfds=1024
minprocs=200

[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[supervisorctl]
serverurl=unix:///tmp/supervisor.sock

[include]
files = ~/supervisor/conf.d/*.conf
EOL

# Create Frappe supervisor config
mkdir -p config/supervisor
cat > config/supervisor/frappe.conf << EOL
[program:frappe-web]
command=bench serve --port 8002
directory=$(pwd)
autostart=true
autorestart=true
stdout_logfile=~/supervisor/logs/frappe-web.log
stderr_logfile=~/supervisor/logs/frappe-web.error.log

[program:frappe-workers]
command=bench worker
directory=$(pwd)
autostart=true
autorestart=true
stdout_logfile=~/supervisor/logs/frappe-workers.log
stderr_logfile=~/supervisor/logs/frappe-workers.error.log

[program:frappe-schedule]
command=bench schedule
directory=$(pwd)
autostart=true
autorestart=true
stdout_logfile=~/supervisor/logs/frappe-schedule.log
stderr_logfile=~/supervisor/logs/frappe-schedule.error.log

[group:frappe]
programs=frappe-web,frappe-workers,frappe-schedule
EOL

# Link Frappe config to supervisor
ln -sf $(pwd)/config/supervisor/frappe.conf ~/supervisor/conf.d/

# Start supervisor
supervisord -c ~/supervisor/supervisord.conf

# Reload supervisor config
supervisorctl reread
supervisorctl update
```

6. Start the development server:
```bash
bench start
# OR use supervisor
supervisorctl start frappe:
```

## Configuration

### Database Configuration
The site is configured to use MariaDB with the following settings:
- Host: 127.0.0.1
- Port: 3306
- Database: healthcare
- User: healthcare
- Password: root

### Redis Configuration
Redis servers run on the following ports:
- Queue: 11002
- Cache: 13002

### Supervisor Configuration
Supervisor manages the following processes:
- frappe-web: The main web server
- frappe-workers: Background job workers
- frappe-schedule: Scheduled tasks
All processes are grouped under the 'frappe' group

## Common Issues and Solutions

### 1. MariaDB SSL Mode Issue
If you encounter SSL mode errors with MariaDB, update the common site configuration (`sites/common_site_config.json`):
```json
{
    "db_host": "127.0.0.1",
    "db_port": 3306,
    "db_ssl": false,
    "db_ssl_ca": null,
    "db_ssl_cert": null,
    "db_ssl_key": null
}
```

### 2. Module Not Found Error
If you get "No module named 'frappe'" error:
```bash
source env/bin/activate
cd apps/frappe
pip install -e .
cd ../..
bench setup requirements
```

### 3. Redis Connection Issues
Make sure Redis is running on the correct ports:
- Queue: 11002 (not 11311)
- Cache: 13002

### 4. Supervisor Issues
If supervisor shows "no such group" error:
```bash
# Check supervisor is running
ps aux | grep supervisord

# If not running, start it:
supervisord -c ~/supervisor/supervisord.conf

# Reload configuration
supervisorctl reread
supervisorctl update

# Check status
supervisorctl status
```

## Development Workflow

1. Start all services:
```bash
supervisorctl start frappe:
```

2. Access the site at:
- http://healthcare.localhost:8002

3. Login with:
- Username: Administrator
- Password: (the one you set during site creation)

4. Monitor processes:
```bash
supervisorctl status
```

## Repository Structure

Important files and directories:
- `apps/` - Contains all Frappe apps (frappe, your custom apps)
- `sites/` - Site-specific files
- `config/` - Configuration files for Redis and other services
- `config/supervisor/` - Supervisor process management configuration
- `env/` - Python virtual environment (not tracked in git)

## Git Configuration

The repository is set up with proper `.gitignore` rules to:
- Track application code and configuration templates
- Ignore sensitive configuration files
- Ignore temporary and generated files
- Ignore user-specific files

### What's tracked:
- Application code
- Configuration templates
- Documentation
- Dependencies list

### What's ignored:
- Sensitive configuration files
- Logs and temporary files
- User uploads
- Virtual environments
- Node modules
- Database files
- Redis dump files

## Contributing

1. Create a new branch for your feature
2. Make your changes
3. Submit a pull request

## License

[Your License Here] 