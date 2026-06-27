# Hello World — a sample Kira extension

A small **toolkit of self-contained tools** that doubles as a tour of the Kira
extension SDK. Every button is a real little task (greet someone, split a bill,
steep some tea…) that happens to exercise a different part of the SDK, so you can
read one action file and see exactly how a feature is used in practice.

Nothing here touches AWS: **every action is `contextMode: "none"`**, so the
extension never asks for an account, a role or a secret — it just runs.

> New to extensions? Start with the
> [extension development guide](https://docs.kira.thiennguyen.dev/extension-development/overview),
> then come back and read these actions as worked examples.

## The actions

| Button | SDK it shows off | What it does |
|---|---|---|
| **Say Hello** | `Ask` (text · number · select) · `Confirm` · `Toast` · `ShowHTML` · `Config` | A 3-step onboarding flow that builds a personalised greeting card. |
| **Sign Up** | `FormInto` (every field kind) · `Toast` · `ShowHTML` | A registration form in one modal, then a summary card. |
| **Split the Bill** | `Form` · `Config` · `Alert` · `Toast` · `ShowHTML` | A restaurant-bill calculator with a per-person breakdown. |
| **Tea Timer** | `Ask` · `ShowProgress`/`UpdateProgress` · `Context` · `Toast` | A cancellable countdown driving the modal progress bar. |
| **Caffeine** | `SetButtonLabel`/`SetButtonProgress` · `ButtonClicks` · `ResetButton` | An in-button countdown — the timer lives in the button itself. |
| **Random Cat Fact** | `HTTP` · `Context` · `Toast` · `ShowHTML` | A real outbound request to a public API, cancel-aware. |
| **About** | `Extension*` / `KiraVersion` metadata · `ShowHTML` | A help card listing every tool and the SDK metadata getters. |

### Say Hello — [`actions/greeting.go`](actions/greeting.go)
Walks through the input modals: `Ask` for text → number → select, gated by a
`Confirm`, with toasts along the way and a final card rendered via `ShowHTML`.
`Ask` has no built-in validation, so the name and age steps **validate in code
and re-prompt** until the value is valid (or the user cancels) — the canonical
pattern. The default name is read from the manifest with `k.Config("defaultName")`.

### Sign Up — [`actions/signup.go`](actions/signup.go)
One `FormInto` call collecting **every field kind**: `text`, `textarea`,
`number`, `secret` (masked), `select`, `radio`, `checkbox` and `multiselect`.
`Required` fields gate the submit. The masked password is **never logged, stored
or shown** — only whether one was entered — demonstrating how to handle a secret
field responsibly.

### Split the Bill — [`actions/billsplit.go`](actions/billsplit.go)
A short `Form`, a bit of arithmetic, and a per-person breakdown via `ShowHTML`.
The currency symbol comes from `k.Config("currency")`; bad input falls back to a
toast plus an `Alert`.

### Tea Timer — [`actions/timer.go`](actions/timer.go)
Picks a duration with `Ask`, then drives the **non-blocking progress modal**
(`ShowProgress` → `UpdateProgress` → `HideProgress`) one tick a second. Each tick
checks `k.Context()` so the modal's Cancel / ✕ aborts promptly. Calls
`HideButtonProgress()` so the button's default run ring doesn't duplicate the
modal.

### Caffeine — [`actions/caffeine.go`](actions/caffeine.go)
The same idea as Tea Timer, but with **no modal** — the countdown lives in the
action button itself: a live `M:SS` label via `SetButtonLabel` and a draining
ring via `SetButtonProgress`. `ButtonClicks()` makes the running button tappable
so a click stops it early; `ResetButton()` restores it on finish/cancel.

### Random Cat Fact — [`actions/catfact.go`](actions/catfact.go)
A genuine outbound `GET` through `k.HTTP()`. The client is bound to the run's
context, so Cancel / timeout aborts the request immediately. The endpoint is
overridable via `k.Config("factUrl")`, and a network hiccup degrades to a
friendly error toast instead of failing hard.

### About — [`actions/about.go`](actions/about.go)
Reads the run's metadata (`Extension`, `ExtensionVersion`, `ExtensionAuthor`,
`KiraVersion`) and renders a card listing what every button does — a one-shot
look at the SDK's metadata getters.

## Manifest config

The extension declares a few values under `config` in
[`manifest.json`](manifest.json); the actions read them with `k.Config(...)`:

| Key | Default | Used by |
|---|---|---|
| `defaultName` | `friend` | Say Hello (prefilled name) |
| `currency` | `$` | Split the Bill (currency symbol) |
| `factUrl` | `https://catfact.ninja/fact` | Random Cat Fact (API endpoint) |

Edit these via **Configure** on the extension tile to change the defaults without
touching the source.

## Layout

```
hello-world/
├── manifest.json        # id, metadata, config, the 7 action definitions
├── icon.svg             # the extension's tile logo
├── actions/             # one Go file per action (package action, func Run)
│   ├── greeting.go
│   ├── signup.go
│   ├── billsplit.go
│   ├── timer.go
│   ├── caffeine.go
│   ├── catfact.go
│   └── about.go
├── assets/              # per-action button glyphs (referenced by manifest `icon`)
└── hello-world.kext     # the packaged bundle, ready to install
```

Each action is plain `package action` with a `func Run(k *kira.Ctx) error`
entry point. Scripts are **interpreted at runtime** — no Go toolchain is needed
on the machine running Kira.

## Install it

- **From the bundle:** in Kira go to **Extensions → Install extension → Choose
  file…** and pick [`hello-world.kext`](hello-world.kext). Review the source at
  the trust gate, then click any action button.
- **Drag & drop:** drop `hello-world.kext` onto the Extensions screen.

Because every action is `contextMode: "none"`, you won't be asked for an account —
just click a button and watch.

## Rebuild the bundle

After editing the manifest or any action, repackage the `.kext` (a plain zip)
from inside this folder:

```sh
cd examples/extensions/hello-world
zip -r -X ../hello-world.kext manifest.json actions assets icon.svg -x '*.DS_Store'
```

## Learn more

- [Extension development guide](https://docs.kira.thiennguyen.dev/extension-development/overview)
- [`docs/EXTENSIONS.md`](../../../docs/EXTENSIONS.md) — bundle format and SDK reference
- [`docs/EXTENSION_DEVELOPMENT.md`](../../../docs/EXTENSION_DEVELOPMENT.md) — full authoring guide
