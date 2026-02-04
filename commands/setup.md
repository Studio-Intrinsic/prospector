---
description: Set up Prospector with API keys for Exa, Apollo, and optionally Attio
allowed-tools:
  - Bash
  - AskUserQuestion
  - Write
  - Read
---

# /prospector:setup

Set up Prospector by collecting and validating API keys.

## Workflow

### Step 1: Check Existing Config

```bash
python3 -c "
from pathlib import Path
config_path = Path.home() / '.config' / 'prospector' / 'config.json'
if config_path.exists():
    import json
    with open(config_path) as f:
        config = json.load(f)
    print('EXISTS')
    print(f'exa: {\"***\" + config.get(\"exa_api_key\", \"\")[-4:] if config.get(\"exa_api_key\") else \"not set\"}')
    print(f'apollo: {\"***\" + config.get(\"apollo_api_key\", \"\")[-4:] if config.get(\"apollo_api_key\") else \"not set\"}')
    print(f'attio: {\"***\" + config.get(\"attio_api_key\", \"\")[-4:] if config.get(\"attio_api_key\") else \"not set\"}')
else:
    print('NOT_FOUND')
"
```

If EXISTS, ask user:
```
header: "Config"
question: "Existing config found. What would you like to do?"
options:
  - label: "Update all keys"
    description: "Re-enter all API keys"
  - label: "Update specific key"
    description: "Update just one key"
  - label: "Keep existing"
    description: "Cancel setup, keep current config"
multiSelect: false
```

If "Update specific key", ask which one, then only collect that key.

### Step 2: Collect Exa API Key

Tell user:
> **Exa API Key**
> Get yours at: https://exa.ai
> 1. Sign up or log in
> 2. Go to API Keys in dashboard
> 3. Create a new key

Ask for the key (user will paste it as "Other" response):
```
header: "Exa"
question: "Paste your Exa API key:"
options:
  - label: "I have it ready"
    description: "I'll paste the key"
multiSelect: false
```

When user provides key, validate it:

```bash
python3 -c "
import httpx
key = '[USER_PROVIDED_KEY]'
try:
    with httpx.Client(timeout=30) as client:
        resp = client.post(
            'https://api.exa.ai/search',
            headers={'x-api-key': key, 'Content-Type': 'application/json'},
            json={'query': 'test', 'numResults': 1},
        )
        if resp.status_code == 200:
            print('VALID')
        else:
            print(f'INVALID: {resp.status_code}')
except Exception as e:
    print(f'ERROR: {e}')
"
```

If INVALID or ERROR, tell user and ask them to try again.

### Step 3: Collect Apollo API Key

Tell user:
> **Apollo API Key**
> Get yours at: https://app.apollo.io/settings/integrations/api
> 1. Log in to Apollo
> 2. Go to Settings → Integrations → API
> 3. Create a new API key

Ask and validate same pattern as Exa:

```bash
python3 -c "
import httpx
key = '[USER_PROVIDED_KEY]'
try:
    with httpx.Client(timeout=30) as client:
        resp = client.post(
            'https://api.apollo.io/api/v1/mixed_people/search',
            headers={'x-api-key': key, 'Content-Type': 'application/json'},
            json={'per_page': 1},
        )
        if resp.status_code == 200:
            print('VALID')
        else:
            print(f'INVALID: {resp.status_code}')
except Exception as e:
    print(f'ERROR: {e}')
"
```

### Step 4: Collect Attio API Key (Optional)

Ask user:
```
header: "Attio"
question: "Do you want to sync leads to Attio CRM?"
options:
  - label: "Yes"
    description: "I use Attio and want to sync leads there"
  - label: "Skip"
    description: "I don't use Attio or will import CSV manually"
multiSelect: false
```

If Yes:
> **Attio API Key**
> Get yours at: https://app.attio.com/settings/developers
> 1. Log in to Attio
> 2. Go to Settings → Developers → API Keys
> 3. Create a new API key with read/write access

Validate:

```bash
python3 -c "
import httpx
key = '[USER_PROVIDED_KEY]'
try:
    with httpx.Client(timeout=30) as client:
        resp = client.get(
            'https://api.attio.com/v2/self',
            headers={'Authorization': f'Bearer {key}'},
        )
        if resp.status_code == 200:
            print('VALID')
        else:
            print(f'INVALID: {resp.status_code}')
except Exception as e:
    print(f'ERROR: {e}')
"
```

### Step 5: Save Config

Save the validated keys:

```bash
python3 -c "
import json
import os
from pathlib import Path

config_path = Path.home() / '.config' / 'prospector' / 'config.json'
config_path.parent.mkdir(parents=True, exist_ok=True)

config = {
    'exa_api_key': '[EXA_KEY]',
    'apollo_api_key': '[APOLLO_KEY]',
    'attio_api_key': '[ATTIO_KEY or null]'
}

with open(config_path, 'w') as f:
    json.dump(config, f, indent=2)

os.chmod(config_path, 0o600)
print(f'Config saved to {config_path}')
print('Permissions set to owner-only (chmod 600)')
"
```

### Step 6: Confirm Success

Tell user:
> Setup complete! Your API keys are saved securely.
>
> Run `/prospector` to find your first leads.

## Notes

- Keys are stored in `~/.config/prospector/config.json`
- File permissions are set to 600 (owner read/write only)
- Attio is optional - you can always import the CSV manually
