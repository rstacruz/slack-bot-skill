# Slack bot setup

## Create Slack app

1. Open <https://api.slack.com/apps>.
2. Click **Create new app** Create a new app for the target workspace.
3. Choose **From manifest**, then paste the manifest below.

```json
{
  "display_information": {
    "name": "Your App Name"
  },
  "features": {
    "bot_user": {
      "display_name": "Your Bot Name",
      "always_online": false
    }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "channels:history",
        "channels:read",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "users:read",
        "chat:write",
        "search:read.public",
        "usergroups:read",
        "usergroups:write",
        "channels:join",
        "channels:manage"
      ]
    }
  },
  "settings": {
    "org_deploy_enabled": false,
    "socket_mode_enabled": false,
    "token_rotation_enabled": false
  }
}
```

Alternatively: if you prefer not using a manifest, go to **OAuth & Permissions**, then add those **Bot Token Scopes** from above.

## Install and configure locally

1. Install the app to the workspace.
2. Copy the **Bot User OAuth Token**. It starts with `xoxb-`.
3. Store it locally as `SLACK_BOT_XOXB_TOKEN`. Here's one way using [Mise](https://mise.jdx.dev):

```bash
# Example: ~/mise.toml if using mise
[env]
SLACK_BOT_XOXB_TOKEN = 'xoxb-...'
```


Reinstall the app after changing scopes.
