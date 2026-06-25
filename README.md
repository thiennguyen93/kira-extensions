# Kira Extensions

A curated collection of extensions for [Kira](https://github.com/your-org/kira).

Extensions add custom actions to Kira — one-click buttons that run a small Go
script with access to your AWS account context, Secrets Manager, HTTP, and
interactive prompts. Each extension in this repo ships as a `.kext` bundle you
can install directly, plus its readable source so you can review before you
trust it.

> ⚠️ **Extensions run code on your machine.** A trusted extension executes
> arbitrary Go with your user's privileges. Only install extensions you have
> read and trust. See [Security](#security).

---

## Repository layout

```
extensions/
  hello-world/            # source for an extension
    manifest.json
    icon.svg
    actions/
      hello.go
      greet.go
      report.go
    hello-world.kext      # built, installable bundle
  trigger/
    ...
```

Each extension lives in its own directory containing the source, and ships a
built `.kext` bundle (a zip) ready to install.

---

## Installing an extension

1. Download the extension's `.kext` file.
2. Open Kira → **Home → Extensions**.
3. **Drag the `.kext` onto the window**, or use **Import** (from file or URL).
4. Kira shows the **trust gate**: review the manifest and the full action source,
   then confirm once. The extension's buttons then appear in the Extensions
   screen.

Extensions declare a `minKiraVersion`; Kira flags any bundle that needs a newer
version than you're running.

---

## What an extension looks like

A bundle is a zip named `*.kext` containing a `manifest.json`, the action source
files it references, and an optional icon:

```
manifest.json
icon.svg
actions/
  staging.go
  production.go
```

### `manifest.json`

```json
{
  "id": "trigger",
  "name": "Trigger",
  "description": "Fire deploy webhooks for staging and production.",
  "version": "1",
  "author": "Your Name",
  "minKiraVersion": "5.0.0",
  "icon": "icon.svg",

  "config": {
    "webhookUrl": "https://example.com/deploy",
    "retry": { "max": 3, "backoffMs": 500 }
  },

  "actions": [
    {
      "id": "staging",
      "label": "Staging",
      "entry": "actions/trigger.go",
      "contextMode": "custom",
      "account": "123456789",
      "role": "deploy-role",
      "region": "us-east-1",
      "config": { "secretKey": "/deploy/staging/token" }
    },
    {
      "id": "production",
      "label": "Production",
      "entry": "actions/trigger.go",
      "contextMode": "inherit",
      "console": "always"
    }
  ]
}
```

| Field            | Level             | Meaning                                                                 |
| ---------------- | ----------------- | ----------------------------------------------------------------------- |
| `id`             | extension, action | Stable identifier. Extension id matches `[A-Za-z0-9._-]+`; action ids are unique within the extension. |
| `name` / `label` | extension / action| Display name and button text.                                           |
| `description`    | extension         | Shown under the name.                                                    |
| `version`        | extension         | Your version string.                                                     |
| `author`         | extension         | Shown in the UI.                                                         |
| `minKiraVersion` | extension         | Minimum Kira version (e.g. `5.0.0`). Empty = no requirement.             |
| `icon`           | extension         | Path inside the bundle to a small logo (`png`/`jpg`/`svg`/`webp`/`gif`). |
| `config`         | both              | Free-form JSON (flat or nested) read by scripts. Per-action keys override extension-level keys. |
| `entry`          | action            | Path inside the bundle to the action's Go file.                         |
| `contextMode`    | action            | Where the AWS account context comes from — see below.                   |
| `account` / `role` / `region` | both | Account context. At extension level it's the default; at action level it's used when `contextMode` is `custom`. |
| `console`        | both              | When the run console opens — see below.                                  |

**`contextMode`**

| Value               | Behaviour                                                              |
| ------------------- | --------------------------------------------------------------------- |
| `inherit` (default) | Use the extension-level `account` / `role` / `region`.                |
| `custom`            | Use this action's own `account` / `role` / `region`.                  |
| `none`              | The action needs no AWS account; Kira never prompts for one.          |

**`console`**

| Value            | Behaviour                                                   |
| ---------------- | ---------------------------------------------------------- |
| `auto` (default) | Open the run console only when the script logs or errors.  |
| `always`         | Always open on run.                                        |
| `none`           | Never auto-open (reopen via the **Output** button).        |

An action's `console` overrides the extension-level default.

### An action

Every action file declares `package action` and a `Run` function. It imports the
`kira` SDK and receives a `*kira.Ctx`:

```go
package action

import "kira"

func Run(k *kira.Ctx) error {
	k.Log("hello from my extension")

	if err := k.Alert("Hello from Kira 👋"); err != nil {
		return err // run was cancelled while the modal was open
	}

	if name := k.Config("greetingName"); name != "" {
		k.Logf("greeting target: %s", name)
	}
	return nil
}
```

One file can back several actions — they share the file and branch on
`k.Action()`:

```go
id, label := k.Action()
if id == "production" {
	ok, err := k.Confirm("Fire the PRODUCTION webhook now?")
	if err != nil || !ok {
		return err
	}
}
```

---

## The `kira` SDK

Every action receives a `*kira.Ctx`. The whole API hangs off it:

| Method | Purpose |
| ------ | ------- |
| `k.Action() (id, label)`              | Which button was clicked. |
| `k.Extension() (id, name)`            | The extension this action belongs to. |
| `k.Account() (account, role, region)` | The resolved account context for this run. |
| `k.Context() context.Context`         | Cancelled on timeout or when the user hits **Stop**. Pass it to long operations. |
| `k.Secret(id)`                        | Resolve a Secrets Manager secret to its raw value, in the run's account. |
| `k.SecretField(id, field)`            | Resolve a JSON secret and return one string field. |
| `k.Config(key)`                       | A config value as a string (objects/arrays come back as JSON). |
| `k.ConfigValue(key) (any, ok)`        | The raw decoded value (maps/arrays for nested config). |
| `k.ConfigRaw()` / `k.ConfigInto(v)`   | Full merged config as JSON / unmarshalled into your struct. |
| `k.Log(...)` / `k.Logf(...)`          | Write to the run console. (`fmt.Println` is captured too.) |
| `k.Alert(msg)`                        | Modal with OK. Pauses the run timeout while open. |
| `k.Confirm(msg) (bool, error)`        | OK/Cancel modal — gate a destructive step. |
| `k.Ask(InputRequest) (val, ok, err)`  | Collect input: `text`, `number`, or `select`. |
| `k.ShowHTML(HTMLReport)`              | Render an HTML report in a modal (print / open in browser). |
| `k.HTTP() *http.Client`               | HTTP client bound to the run context, so **Stop** aborts in-flight requests. |

Notes:

- **Config merge** — for a run, the action's `config` is merged over the
  extension's `config`; action keys win.
- **Secrets are resolved in the run's account context** — pick the account with
  `contextMode` and the `account` / `role` / `region` fields. Raw secret values
  stay on your machine; only your script decides what to do with them.
- **Cancellation** — pass `k.Context()` to anything slow; honour it between
  retries and during backoff so **Stop** and the run timeout work cleanly.

A fuller worked example (retries, backoff, secret-backed webhook POST) lives in
[`extensions/trigger`](extensions/trigger).

---

## Building a `.kext`

A `.kext` is just a zip of the bundle's contents (the `manifest.json` must be at
the root):

```bash
cd extensions/hello-world
zip -r ../hello-world.kext manifest.json icon.svg actions/
```

Kira accepts both `.kext` and `.zip`. Bundles are capped (5 MB compressed,
200 files, 2 MB per file) — keep them lean.

---

## Encrypted bundles (optional)

You can password-lock a bundle so installers must enter a password before Kira
will unlock, review, and install it. Either use the in-app **Encrypt a bundle**
tool, or the CLI:

```bash
KEXT_PASSWORD='your-password' kext-encrypt hello-world.kext hello-world.locked.kext
```

The installer is prompted for the password on import; the source is only
revealed in the trust gate after it unlocks.

---

## Security

Kira extensions are powerful by design: a trusted extension runs **arbitrary
Go with your user's privileges** — full filesystem, network, and AWS access via
your account context.

The trust model is review-then-confirm:

- Kira **shows you the full action source** and manifest in a trust gate.
- You **confirm once**; nothing runs until you do.
- Kira **refuses to run any action from an extension you haven't trusted.**

Treat installing an extension like running a script someone handed you. Read the
code. Only install from sources you trust. Encrypting a bundle protects it in
transit — it does **not** make the code safe to run blindly.

---

## Contributing

1. Add a directory under `extensions/` with your `manifest.json`, action
   source, and an optional icon.
2. Keep actions small and readable — they're meant to be reviewed at the trust
   gate.
3. Build and commit the `.kext` alongside the source.
4. Document what the extension does, which AWS context it needs, and any config
   keys in a short README in its directory.
5. Open a pull request.

---

## License

<!-- TODO: choose a license (MIT / Apache-2.0 / proprietary) and add a LICENSE file. -->
