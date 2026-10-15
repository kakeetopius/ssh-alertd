# ssh-alertd

[![CI](https://github.com/Amekiri-Studio/ssh-alertd/actions/workflows/ci.yml/badge.svg)](https://github.com/Amekiri-Studio/ssh-alertd/actions/workflows/ci.yml)

A small, modular SSH alert daemon written in Go. It watches `sshd` logs and, on every successful SSH login, sends an alert containing:

- **IP**: the client's source IP
- **Username**: the login username
- **Time**: when the login happened
- **Client port**: the client's source port, taken from the sshd log line `from <IP> port <port>` (not the server's listening port `22`)

Telegram and SMTP are implemented today. The notifier layer is an interface, so WhatsApp, WeCom, DingTalk and Feishu can be added as self-contained files without touching the rest of the system.

## Architecture

```text
main.go                       wiring: config -> source -> monitor -> dispatcher
internal/
  event/event.go              LoginEvent domain model
  config/config.go            JSON config loading + validation
  monitor/
    source.go                 Source interface; journald & file followers
    monitor.go                sshd line parser -> LoginEvent
  notifier/
    notifier.go               Notifier interface + concurrent Dispatcher
    telegram.go               Telegram Bot backend (one file per backend)
    smtp.go                   SMTP (email) backend
```

Data flow: `Source` (journald/file) → `Monitor` parses `Accepted ...` lines → `Dispatcher` fans each `LoginEvent` out to all enabled `Notifier`s concurrently.

## Install

### Arch Linux (AUR)

- [`ssh-alertd`](https://aur.archlinux.org/packages/ssh-alertd): stable release
- [`ssh-alertd-git`](https://aur.archlinux.org/packages/ssh-alertd-git): latest git

```sh
yay -S ssh-alertd        # or: paru -S ssh-alertd
```

### Debian / Ubuntu

Download the `.deb` for your architecture from the [latest release](https://github.com/Amekiri-Studio/ssh-alertd/releases/latest):

```sh
sudo apt install ./ssh-alertd_*_amd64.deb
```

### Other Linux

Download a standalone `linux-amd64` or `linux-arm64` tarball from the [releases page](https://github.com/Amekiri-Studio/ssh-alertd/releases/latest), or build from source (see below).

## Build

```sh
go build -o ssh-alertd .
go test ./...
```

## Releases

Pushing a `v*` tag triggers the [Release workflow](.github/workflows/release.yml), which builds and attaches the following to the GitHub Release:

- `.deb` packages for `amd64` and `arm64`
- Standalone `linux-amd64` and `linux-arm64` tarballs (binary, unit file and example config)
- A `SHA256SUMS` checksum file

```sh
git tag -a v0.1.3 -m "v0.1.3" && git push --follow-tags
```

Keep the version in `packaging/archlinux/PKGBUILD` and `packaging/debian/changelog` in step with the tag.

## Configure

Copy `config.example.json` to `/etc/ssh-alertd/config.json` and fill in your Telegram bot token and chat ID.

- `log_source.type`: `journald` (default, reads sshd via `journalctl -f`) or `file` (tails `log_source.path` via `tail -F`, e.g. `/var/log/auth.log` on Debian/Ubuntu or `/var/log/secure` on RHEL).
- `hostname`: optional override; defaults to the OS hostname.

### Telegram bot setup

Three steps: create the bot to get a token, get the chat ID, then fill in `config.json`. A fourth step verifies the result.

#### 1. Create the bot and get `bot_token`

1. In Telegram, search for the official `@BotFather` and start a chat.
2. Send `/newbot` and provide a display name and a username. The username must end in `bot`, e.g. `my_ssh_alert_bot`.
3. BotFather returns a token like `123456789:AAExxxxxxxxxxxxxxxxxxxxxxxxxx`. That is your `bot_token`.

#### 2. Get `chat_id`

A bot cannot message you first; you must message it before it can learn the chat ID.

- Private chat: search for your bot, tap **Start**, then send any message.
- Group (for multiple recipients): add the bot to the group and send a message there.

Then open this URL in a browser, replacing `<TOKEN>` with your token:

```text
https://api.telegram.org/bot<TOKEN>/getUpdates
```

Find `"chat":{"id":...}` in the returned JSON. A private chat ID is a positive number (e.g. `987654321`); a group ID is negative (supergroups often start with `-100`, e.g. `-1001234567890`). If a group returns no updates, disable privacy mode with `/setprivacy` in BotFather and send the message again.

#### 3. Fill in `config.json`

```json
"telegram": {
  "enabled": true,
  "bot_token": "123456789:AAExxxxxxxxxxxxxxxxxxxxxxxxxx",
  "chat_id": "-1001234567890",
  "api_base": "https://api.telegram.org"
}
```

- `enabled` must be `true`, otherwise the backend is not registered and startup exits with `no notifiers enabled`.
- `bot_token` and `chat_id` are both strings, so keep the quotes. An empty value for either fails startup.
- `api_base` is usually left at the default. Change it only when you need a reverse proxy (networks where `api.telegram.org` is not directly reachable).

#### 4. Verify

Send one message by hand first to confirm the token and chat ID are correct:

```sh
curl -s "https://api.telegram.org/bot<TOKEN>/sendMessage" \
  -d chat_id=<CHAT_ID> -d text="ssh-alertd test"
```

`"ok":true` means the config is correct, and you will receive an alert on the next real SSH login.

> **Warning:** `config.json` contains the bot token. Set `chmod 600` and keep it out of git.

#### Custom message (optional)

By default the message uses a built-in HTML format. To customize the text, set `message_template`: a [Go template](https://pkg.go.dev/text/template) with the same event fields as SMTP (`.Username`, `.IP`, `.Port`, `.Method`, `.Hostname`, `.Time`).

```json
"telegram": {
  "enabled": true,
  "bot_token": "123456789:AAExampleBotTokenReplaceMe",
  "chat_id": "-1001234567890",
  "message_template": "🔐 <b>{{.Username}}</b> logged in from <code>{{.IP}}</code> on {{.Hostname}}",
  "parse_mode": "HTML"
}
```

- `message_template`: empty (default) uses the built-in HTML format.
- `message_template_file`: a path read as the message template. It takes precedence over `message_template` and is handy for multi-line messages.
- `parse_mode`: `HTML` (default), `MarkdownV2`, `Markdown`, or `none` (plain text). With `HTML`, event fields are auto-escaped (via `html/template`) so a value such as a username cannot break the markup. With the other modes you are responsible for any escaping the format requires.

> **Note:** Telegram's HTML supports only a small set of inline tags (`<b>`, `<i>`, `<code>`, `<pre>`, `<a>`, and similar). There is no layout or color, so keep templates to formatted text.

Ready-to-use examples live in [`examples/telegram/`](examples/telegram/).

### SMTP (email) setup

Add an `smtp` block under `notifiers` to receive alerts by email. This is handy on networks where Telegram is unreachable, since it talks directly to your own mail server.

```json
"smtp": {
  "enabled": true,
  "host": "smtp.example.com",
  "port": 587,
  "username": "alert@example.com",
  "password": "your-smtp-password",
  "from": "alert@example.com",
  "to": ["admin@example.com"],
  "encryption": "starttls"
}
```

- `encryption`: `starttls` (default, typically port `587`), `tls` (implicit TLS / SMTPS, typically port `465`), or `none` (port `25`, no transport security).
- `port`: optional; defaults to `465` for `tls`, otherwise `587`.
- `username` / `password`: optional. Omit `username` to skip authentication (e.g. an internal relay). When set, `PLAIN` auth is used, so always use it over TLS.
- `to`: one or more recipients; at least one is required.
- Multiple notifiers can be enabled at once. Each enabled backend receives every alert independently.

Telegram and SMTP are independent: enabling one does not require the other.

#### Custom email templates

The subject and body can be customized with [Go templates](https://pkg.go.dev/text/template). Templates receive the login event with these fields:

| Field       | Example                                                             |
| ----------- | ------------------------------------------------------------------- |
| `.Username` | `alice`                                                             |
| `.IP`       | `203.0.113.5`                                                       |
| `.Port`     | `50568` (client source port)                                        |
| `.Method`   | `publickey` / `password`                                            |
| `.Hostname` | `web-01`                                                            |
| `.Time`     | a `time.Time`; format with `{{.Time.Format "2006-01-02 15:04:05"}}` |

```json
"smtp": {
  "enabled": true,
  "host": "smtp.example.com",
  "from": "alert@example.com",
  "to": ["admin@example.com"],
  "subject_template": "[ALERT] SSH login {{.Username}}@{{.Hostname}}",
  "body_template": "{{.Username}} logged in from {{.IP}}:{{.Port}} via {{.Method}} at {{.Time.Format \"2006-01-02 15:04:05\"}}",
  "html": false
}
```

- `subject_template` / `body_template`: inline Go templates. Empty values use the built-in subject and body.
- `body_template_file`: a path read as the body template. It takes precedence over `body_template` and is handy for multi-line or HTML bodies.
- `html`: when `true`, renders the body as `text/html` (using `html/template`, which auto-escapes the event fields). Pair it with a `body_template`.

Templates are compiled at startup, so a malformed template fails fast with a clear error rather than silently dropping alerts.

Ready-to-use HTML and plain-text examples live in [`examples/email/`](examples/email/).

## Run

```sh
sudo ./ssh-alertd -config /etc/ssh-alertd/config.json
```

### As a systemd service

```sh
sudo install -m0755 ssh-alertd /usr/local/bin/ssh-alertd
sudo install -d /etc/ssh-alertd
sudo install -m0640 config.example.json /etc/ssh-alertd/config.json   # then edit
sudo install -m0644 deploy/ssh-alertd.service /etc/systemd/system/ssh-alertd.service

# Create the dedicated ssh-alertd user and give it ownership of the config.
sudo install -Dm644 deploy/ssh-alertd.sysusers /usr/lib/sysusers.d/ssh-alertd.conf
sudo install -Dm644 deploy/ssh-alertd.tmpfiles /usr/lib/tmpfiles.d/ssh-alertd.conf
sudo systemd-sysusers
sudo systemd-tmpfiles --create

sudo systemctl daemon-reload
sudo systemctl enable --now ssh-alertd
```

The service runs as the dedicated `ssh-alertd` user, which owns the config and belongs to the `systemd-journal` group for the journald source. For the `file` source, also grant it read access to the auth log (see [Privileges](#privileges)).

## Privileges

ssh-alertd does not require full root. It does not listen on a port or write system files; it only reads sshd login logs and makes HTTPS requests to Telegram. The privileges needed depend entirely on how the log is read, and reading SSH authentication logs is a protected operation on Linux.

| Log source (`log_source.type`) | Reads                                                         | Privileges required                                                            |
| ------------------------------ | ------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `journald` (default)           | systemd journal                                               | root, or membership in the `systemd-journal` group                             |
| `file`                         | `/var/log/auth.log` (Debian/Ubuntu), `/var/log/secure` (RHEL) | root, or a group with read access to the file (usually `adm` on Debian/Ubuntu) |

Confirm access with the matching command first. If you can see output, the daemon can read it too:

```sh
journalctl -t sshd -t sshd-session -n5    # journald source
tail -n5 /var/log/auth.log                # file source
```

### systemd deployment

The unit runs as the dedicated system user `ssh-alertd` (created from `deploy/ssh-alertd.sysusers` by `systemd-sysusers`) and is granted read access to the journal via `SupplementaryGroups=systemd-journal`, so the journald source needs no extra authorization when started via systemd. That user also takes ownership of `/etc/ssh-alertd/config.json` via `deploy/ssh-alertd.tmpfiles`, so it can read the token. The file stays `0640`, unreadable to other local users.

> **Note:** The Arch and Debian packages install these two files and trigger sysusers/tmpfiles automatically. For a manual systemd deployment, run once:
>
> ```sh
> sudo install -Dm644 deploy/ssh-alertd.sysusers /usr/lib/sysusers.d/ssh-alertd.conf
> sudo install -Dm644 deploy/ssh-alertd.tmpfiles /usr/lib/tmpfiles.d/ssh-alertd.conf
> sudo systemd-sysusers && sudo systemd-tmpfiles --create
> ```

For the `file` source, the `ssh-alertd` user cannot read `auth.log` by default. Add the log file's group to the unit, e.g. on Debian/Ubuntu:

```ini
SupplementaryGroups=systemd-journal adm
```

### Manual debugging

The simplest approach is to run as root directly:

```sh
sudo ./ssh-alertd -config /etc/ssh-alertd/config.json
```

Or add your current user to the relevant group to avoid `sudo` (log in again for the group to take effect):

```sh
sudo usermod -aG systemd-journal $USER   # journald source
sudo usermod -aG adm $USER               # file source (Debian/Ubuntu)
```

## Troubleshooting

ssh-alertd writes diagnostics to stderr (view them with `journalctl -u ssh-alertd -f` under systemd). A healthy start shows `enabled notifier: telegram` and `monitor started on source ...`. Every successful login produces `detected SSH login: ...` and `notifier telegram delivered alert ...`.

| Symptom                                                         | Likely cause                       | Fix                                                                                                             |
| --------------------------------------------------------------- | ---------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Exits at startup with `no notifiers enabled`                    | `telegram.enabled` is not `true`   | Set it to `true`                                                                                                |
| Startup error `telegram enabled but bot_token/chat_id is empty` | Missing credentials                | Fill in `bot_token` and `chat_id` (strings)                                                                     |
| Startup error `parse config ...`                                | JSON syntax error or unknown field | The config uses `DisallowUnknownFields`; remove typos and extra keys, and compare against `config.example.json` |
| No `detected` log after a login                                 | Log source cannot read sshd        | See [No log lines](#no-log-lines-no-detected)                                                                   |
| `detected` present but `notifier telegram failed`               | Telegram delivery failed           | See [No Telegram message](#no-telegram-message-notifier-telegram-failed-in-the-log)                             |

### No log lines (no `detected`)

1. Confirm access first. If you can see output, the daemon can read it (see [Privileges](#privileges)):

   ```sh
   journalctl -t sshd -t sshd-session -n5    # journald source
   tail -n5 /var/log/auth.log                # file source
   ```

2. Confirm sshd is actually logging logins: run `ssh localhost` once and check whether `Accepted password/publickey for ...` appears in the output of the command above.
3. For the `file` source, check that `log_source.path` is correct (`/var/log/auth.log` on Debian/Ubuntu, `/var/log/secure` on RHEL/CentOS).
4. On a few distros sshd's syslog identifier is neither `sshd` nor `sshd-session`, so the journald source filters it out. Switch to the `file` source.

### No Telegram message (`notifier telegram failed` in the log)

The error includes the Telegram API response. Common cases:

- `status 401` / `404`: wrong `bot_token`. Re-check it with BotFather.
- `status 400 ... chat not found`: wrong `chat_id`, or you never messaged the bot. Start or message the bot, then fetch the ID again via `getUpdates`. A group ID is negative, so do not drop the `-100` prefix.
- `status 403 ... bot was blocked`: you blocked the bot. Unblock it.
- `send request: ... timeout / no such host`: the server cannot reach `api.telegram.org` (restricted network). Set `api_base` to your reverse proxy.

A quick `curl` test separates a credential problem from a program problem:

```sh
curl -s "https://api.telegram.org/bot<TOKEN>/sendMessage" \
  -d chat_id=<CHAT_ID> -d text="ssh-alertd test"
```

`"ok":true` means the credentials are fine. On `"ok":false`, the `description` field explains why.

## Adding a new notifier

1. Create `internal/notifier/<backend>.go` implementing the `Notifier` interface (`Name()` and `Send(ctx, event.LoginEvent) error`).
2. Add its config struct to `internal/config/config.go`.
3. Register it in `buildNotifiers` in `main.go`.

## License

Licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for the full text.
