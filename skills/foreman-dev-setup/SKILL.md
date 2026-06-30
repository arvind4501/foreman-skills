---
name: foreman-dev-setup
description: "Automated Foreman development environment setup. Detects OS, checks prerequisites, installs dependencies, configures database, and verifies the setup — all with interactive approval gates."
trigger: /foreman-dev-setup
---

# /foreman-dev-setup

Automate the full Foreman development environment setup from zero to running app. The agent detects what's already installed, skips completed steps, handles errors conversationally, and asks for approval before any destructive or sudo operation.

## Usage

```
/foreman-dev-setup                    # full setup from scratch
/foreman-dev-setup --check            # audit current environment, report what's missing
/foreman-dev-setup --fix              # only run steps that are broken/missing
/foreman-dev-setup --db-only          # just database setup (create, migrate, seed)
/foreman-dev-setup --plugins          # install and configure plugins after base setup
```

## Behavior

When invoked, follow this exact protocol. Each phase gates on user approval before running sudo/install commands. Report results after each step — never go silent.

### Phase 0: Environment Audit

Run ALL of the following checks in parallel and build a status report:

```bash
# OS detection
cat /etc/os-release 2>/dev/null | grep -E '^(ID|VERSION_ID)='

uname -s

# Ruby
ruby --version 2>/dev/null
rbenv --version 2>/dev/null
rvm --version 2>/dev/null

# Node
node --version 2>/dev/null
nvm --version 2>/dev/null
npm --version 2>/dev/null

# PostgreSQL
pg_isready 2>/dev/null
psql --version 2>/dev/null
sudo -u postgres psql -c "SELECT 1" 2>/dev/null

# Build tools
gcc --version 2>/dev/null
make --version 2>/dev/null
git --version 2>/dev/null

# Foreman-specific
ls config/settings.yaml config/database.yml 2>/dev/null
bundle --version 2>/dev/null
```

Read `.github/matrix.json` from the foreman repo to get required versions:
- Ruby version (currently 3.0)
- Node version (currently 22)
- PostgreSQL version (currently 13)

Present a table like:

```
Component     Required   Found        Status
─────────────────────────────────────────────
OS            Fedora/EL  Fedora 44    OK
Ruby          3.0.x      3.0.7        OK
Node          22.x       (none)       MISSING
PostgreSQL    13+        16.1         OK
Bundler       any        2.4.1        OK
npm           any        (none)       MISSING
gcc           any        14.2         OK
git           any        2.47         OK
Foreman repo  cloned     /home/...    OK
settings.yaml exists     (none)       MISSING
database.yml  exists     (none)       MISSING
```

If `--check` flag: stop here, just report.

### Phase 1: System Dependencies

Based on OS detection:

**Fedora:**
```bash
sudo dnf install gcc-c++ make git ruby ruby-devel rubygem-bundler \
    libvirt-devel postgresql-devel openssl-devel \
    libxml2-devel libxslt-devel zlib-devel \
    readline-devel systemd-devel tar libcurl-devel nodejs
```

**EL9 (CentOS/RHEL/Rocky/Alma):**
```bash
# Enable CRB repo for libvirt-devel
sudo dnf config-manager --set-enabled crb
sudo dnf module enable nodejs:22
sudo dnf -y install gcc-c++ make git ruby ruby-devel rubygem-bundler \
    libvirt-devel postgresql-devel openssl-devel \
    libxml2-devel libxslt-devel zlib-devel \
    readline-devel systemd-devel tar libcurl-devel nodejs
```

**macOS:**
```bash
brew install gcc make git libvirt postgresql openssl \
    libxml2 libxslt readline curl node
```

IMPORTANT: Before running any `sudo` command, show the exact command and ask for user approval.

Skip packages that are already installed at the required version.

### Phase 2: Ruby Setup

1. Check if Ruby version matches `.github/matrix.json` requirement
2. If not, install via rbenv:

```bash
# Fedora
sudo dnf install rbenv ruby-build-rbenv
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
source ~/.bashrc
rbenv install 3.0.7
rbenv local 3.0.7
```

3. Verify: `ruby --version` matches required version

### Phase 3: Node Setup

1. Check if Node version matches `.github/matrix.json` requirement
2. If not, install via nvm:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
nvm install 22
nvm use 22
```

3. Verify: `node --version` shows v22.x

### Phase 4: PostgreSQL Setup

1. Check if PostgreSQL is running: `pg_isready`
2. If not installed/running:

```bash
sudo dnf install postgresql-server
sudo postgresql-setup --initdb
sudo systemctl enable --now postgresql
```

3. Create dev user:

```bash
sudo -u postgres createuser --createdb $USER
```

4. Verify: `psql -c "SELECT 1" postgres` succeeds

### Phase 5: Repository Configuration

1. Check if we're in a foreman repo (look for `app/models/host.rb` or similar)
2. If not, ask user for fork URL or use upstream:

```bash
git clone https://github.com/{USER}/foreman.git -b develop
cd foreman
git remote add upstream git@github.com:theforeman/foreman.git
```

3. Copy config templates:

```bash
cp config/settings.yaml.example config/settings.yaml
cp config/database.yml.example config/database.yml
```

4. Ask user if they want to customize `database.yml` (username, password, host)

### Phase 6: Dependency Installation

```bash
bundle install
npm install
```

**Error Recovery:**
- If `pg` gem fails: detect `pg_config` location, retry with `--with-pg-config=<path>`
- If `ruby-libvirt` fails: try `export PKG_CONFIG_PATH=/usr/lib64/pkgconfig` and retry
- If npm peer dep warnings: suggest `--legacy-peer-deps` for npm 7+

### Phase 7: Database Setup

```bash
bundle exec rake db:create
bundle exec rake db:migrate
SEED_ADMIN_PASSWORD=changeme bundle exec rake db:seed
```

Ask user for preferred admin password before seeding. Default: `changeme`

Also set up test database:
```bash
RAILS_ENV=test bundle exec rake db:create
RAILS_ENV=test bundle exec rake db:migrate
```

### Phase 8: Verification

Run these checks and report pass/fail:

```bash
# Rails boots
bundle exec rails runner 'puts "Rails OK: #{Rails.env}"'

# Webpack compiles
bundle exec rake webpack:compile

# Tests pass (quick smoke)
bundle exec rake test TEST=test/unit/host_test.rb 2>&1 | tail -5

# Integration readiness
which chromedriver 2>/dev/null && echo "ChromeDriver: OK" || echo "ChromeDriver: MISSING (needed for integration tests only)"
```

### Phase 9: First Run (Optional)

Ask user if they want to start Foreman:

```bash
bundle exec foreman start
# OR split mode:
# Terminal 1: bundle exec rails s -b 0.0.0.0 -p 3000
# Terminal 2: bundle exec foreman start webpack
```

Monitor first 30 seconds of logs for common errors:
- Port 3000 in use → suggest `lsof -i :3000` and kill
- Missing environment variable → suggest `.env` file
- Migration pending → run `db:migrate`

## Error Recovery Playbook

The agent should handle these common failures automatically:

| Error | Detection | Fix |
|-------|-----------|-----|
| `pg` gem install fails | `extconf.rb` error in bundle output | Find `pg_config`, reinstall with `--with-pg-config` |
| `ruby-libvirt` fails | `libvirt library not found` | `export PKG_CONFIG_PATH=/usr/lib64/pkgconfig` + retry |
| npm peer deps | `ERESOLVE` in npm output | Retry with `--legacy-peer-deps` |
| PostgreSQL auth fails | `Peer authentication failed` | Edit `pg_hba.conf` to use `trust` or `md5` for local |
| Port conflict | `Address already in use` | `lsof -i :<port>`, offer to kill process |
| Wrong Ruby version | Version mismatch in audit | Install correct version via rbenv |
| DB already exists | `already exists` in rake output | Skip, not an error |
| Permission denied on gem install | `EACCES` or `Permission denied` | Check `GEM_HOME`, suggest `--path vendor/bundle` |

## Plugin Setup (--plugins flag)

When `--plugins` is passed or user requests plugin setup:

1. Ask which plugin(s) to install
2. For each plugin:

```bash
# From source (if local path exists)
echo "gem '<PLUGIN_NAME>', path: '../<PLUGIN_PATH>'" >> bundler.d/<PLUGIN_NAME>.local.rb

# From GitHub
echo "gem '<PLUGIN_NAME>', git: 'https://github.com/theforeman/<PLUGIN_NAME>.git'" >> bundler.d/<PLUGIN_NAME>.local.rb
```

3. Run:
```bash
bundle install
npm install  # triggers postinstall for plugin node modules
bundle exec rake db:migrate
bundle exec rake db:seed
```

## Resumability

If the setup is interrupted, re-running `/foreman-dev-setup` should:
1. Re-run Phase 0 audit
2. Skip all phases that are already complete
3. Resume from the first incomplete phase
4. This is achieved by the audit — no state file needed

## Important Rules

- NEVER run `sudo` commands without showing the command and getting user approval first
- NEVER modify system-wide Ruby/Node installations — always use version managers (rbenv/nvm)
- NEVER store passwords in files — ask interactively and use environment variables
- ALWAYS read `.github/matrix.json` for version requirements — never hardcode versions
- If ANY phase fails, stop and troubleshoot before proceeding — don't skip and continue
- Present each phase result clearly before moving to the next
