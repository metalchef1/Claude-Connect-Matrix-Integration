# /matrix:configure

Configure the Matrix channel bot credentials.

## No arguments — show status

Display current configuration:
- Whether MATRIX_HOMESERVER_URL is set (show value)
- Whether MATRIX_ACCESS_TOKEN is set (show masked: first 8 chars + …)
- Whether MATRIX_ROOM_ID is set (show value)
- Whether MATRIX_USER_ID is set (show value)
- Current access policy and allowFrom list (read from ~/.claude/channels/matrix/access.json)
- Location of env file: ~/.claude/channels/matrix/.env

## With arguments — save credentials

Accepted forms:
- `/matrix:configure` — show status only
- `/matrix:configure homeserver=https://... token=... room=!xxx:server user=@bot:server` — set all at once
- `/matrix:configure token=<token>` — update just the token

For each provided key=value pair, write/update the corresponding line in ~/.claude/channels/matrix/.env:
- `homeserver=` → `MATRIX_HOMESERVER_URL=`
- `token=` → `MATRIX_ACCESS_TOKEN=`
- `room=` → `MATRIX_ROOM_ID=`
- `user=` → `MATRIX_USER_ID=`

Steps:
1. Create ~/.claude/channels/matrix/ if missing (mode 0700)
2. Read existing .env if present
3. Update/add relevant lines
4. Write back with mode 0600
5. Show updated status

## `clear` argument — remove stored credentials

Delete ~/.claude/channels/matrix/.env and confirm.
