# djangocommand

Python client for [DjangoCommand](https://djangocommand.com) - run, schedule, and audit Django management commands without SSH access.

## Installation

```bash
pip install djangocommand
```

## Quick Start

1. Add to your `INSTALLED_APPS`:

```python
INSTALLED_APPS = [
    # ...
    'djangocommand',
]
```

2. Add your API key to `settings.py`:

```python
DJANGOCOMMAND_API_KEY = "dc_your_api_key_here"
```

3. Start the runner:

```bash
python manage.py djangocommand start
```

The runner will connect to DjangoCommand, sync your available commands, and start polling for executions.

## Configuration

### Required

```python
# Your project's API key (get this from the DjangoCommand dashboard)
DJANGOCOMMAND_API_KEY = "dc_..."
```

### Optional

```python
# Server URL (default: https://app.djangocommand.com)
DJANGOCOMMAND_SERVER_URL = "https://app.djangocommand.com"

# Runner heartbeat interval in seconds (default: 30, minimum: 5)
DJANGOCOMMAND_HEARTBEAT_INTERVAL = 30

# HTTP request timeout in seconds (default: 30)
DJANGOCOMMAND_REQUEST_TIMEOUT = 30

# Max retries for failed requests (default: 3)
DJANGOCOMMAND_MAX_RETRIES = 3

# Hosts allowed to use HTTP instead of HTTPS (default: localhost only)
DJANGOCOMMAND_ALLOW_HTTP_HOSTS = ['localhost', '127.0.0.1', '::1']
```

## Command Security

The runner includes a security layer that controls which commands can be executed remotely. This protects against accidental or malicious execution of dangerous commands.

### Default: Allowlist Mode (Recommended)

By default, DjangoCommand uses an **allowlist approach** - only commands explicitly listed can be executed remotely. This is the most secure configuration.

The default allowlist includes common, safe Django commands:

| Category | Commands |
|----------|----------|
| Database | `migrate`, `showmigrations`, `dbbackup`, `dbrestore`, `createcachetable` |
| Static files | `collectstatic`, `findstatic` |
| Maintenance | `clearsessions`, `check`, `sendtestemail`, `diffsettings` |
| Data | `dumpdata`, `loaddata`, `inspectdb` |
| Testing | `test` |
| django-extensions | `show_urls`, `validate_templates`, `print_settings`, `clear_cache` |
| Wagtail | `fixtree`, `publish_scheduled`, `update_index` |
| And more... | See `DEFAULT_ALLOWED_COMMANDS` for the full list |

Commands not in the allowlist are:
- **Not synced** to the server (won't appear in the UI)
- **Rejected at runtime** with an error message (defense in depth)

### Adding Commands to the Allowlist

To allow additional commands, extend the default allowlist:

```python
from djangocommand import DEFAULT_ALLOWED_COMMANDS

DJANGOCOMMAND_ALLOWED_COMMANDS = DEFAULT_ALLOWED_COMMANDS + (
    'my_custom_command',
    'another_safe_command',
)
```

### Using a Custom Allowlist

For maximum control, define your own allowlist from scratch:

```python
# Only these 3 commands can be executed remotely
DJANGOCOMMAND_ALLOWED_COMMANDS = (
    'migrate',
    'collectstatic',
    'clearsessions',
)
```

### Removing Commands from the Default Allowlist

Use list comprehension to remove specific commands:

```python
from djangocommand import DEFAULT_ALLOWED_COMMANDS

# Remove 'loaddata' from the allowlist
DJANGOCOMMAND_ALLOWED_COMMANDS = tuple(
    cmd for cmd in DEFAULT_ALLOWED_COMMANDS
    if cmd != 'loaddata'
)
```

### Alternative: Blocklist Mode

If you prefer to allow all commands except dangerous ones (less secure but more permissive), enable blocklist mode:

```python
# Enable blocklist mode
DJANGOCOMMAND_USE_BLOCKLIST = True
```

In blocklist mode, the default blocklist includes:

| Category | Commands |
|----------|----------|
| Database destruction | `flush`, `sqlflush`, `reset_db` |
| Interactive shells | `shell`, `shell_plus`, `dbshell` |
| Development servers | `runserver`, `runserver_plus`, `testserver` |
| Security sensitive | `createsuperuser`, `changepassword` |
| File modifications | `makemigrations`, `squashmigrations` |
| Other dangerous | `drop_test_database`, `delete_squashed_migrations`, `clean_pyc` |

You can extend the blocklist:

```python
DJANGOCOMMAND_USE_BLOCKLIST = True

from djangocommand import DEFAULT_DISALLOWED_COMMANDS

DJANGOCOMMAND_DISALLOWED_COMMANDS = DEFAULT_DISALLOWED_COMMANDS + (
    'my_dangerous_command',
)
```

Or allow a blocked command by removing it:

```python
DJANGOCOMMAND_USE_BLOCKLIST = True

from djangocommand import DEFAULT_DISALLOWED_COMMANDS

DJANGOCOMMAND_DISALLOWED_COMMANDS = tuple(
    cmd for cmd in DEFAULT_DISALLOWED_COMMANDS
    if cmd != 'createsuperuser'
)
```

## Running the Runner

### Foreground (development)

```bash
python manage.py djangocommand start
```

### Background (production)

Use a process manager like systemd or supervisor:

```ini
# /etc/supervisor/conf.d/djangocommand.conf
[program:djangocommand]
command=/path/to/venv/bin/python manage.py djangocommand start
directory=/path/to/your/project
user=www-data
autostart=true
autorestart=true
```

### Docker

```dockerfile
CMD ["python", "manage.py", "djangocommand", "start"]
```

## Programmatic Usage

You can also use the client programmatically:

```python
from djangocommand import Runner

# Create runner from Django settings
runner = Runner.from_settings()

# Run the runner (blocks until stopped)
runner.run()

# Or just run a single heartbeat cycle
runner.run_once()
```

### Checking Command Permissions

```python
from djangocommand import (
    is_command_allowed,
    is_using_blocklist,
    get_allowed_commands,
    get_disallowed_commands,
    DEFAULT_ALLOWED_COMMANDS,
)

# Check if a command is allowed
allowed, reason = is_command_allowed('migrate')
# (True, '')

allowed, reason = is_command_allowed('shell')
# (False, 'not in DJANGOCOMMAND_ALLOWED_COMMANDS allowlist')

# Check which mode is active
if is_using_blocklist():
    blocked = get_disallowed_commands()  # frozenset of command names
else:
    allowed = get_allowed_commands()  # frozenset of command names
```

## License

MIT
