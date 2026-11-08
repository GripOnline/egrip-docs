# e-Grip 4 documentation

Generic documentation for building, maintaining and upgrading e-Grip 4 implementations.
The repository contains Markdown files only and is distributed as a Composer package, so every
client implementation can carry an up-to-date copy of the docs as a (dev) dependency.

## Contents

| Document | Description |
|---|---|
| [docs/upgrade-egrip3-to-egrip4.md](docs/upgrade-egrip3-to-egrip4.md) | Upgrading an e-Grip 3.x client branch to e-Grip 4.0 (also usable as instructions for AI coding agents) |

## Installing in a client implementation

The package is published on the Grip Composer channel:

```json
{
    "repositories": [
        { "type": "composer", "url": "https://composer.hosting.grip.nl/" }
    ],
    "require-dev": {
        "grip/egrip4-docs": "*"
    }
}
```

After `composer update` the docs are available in `vendor/grip/egrip4-docs/docs/`.
Point people and agents there, for instance from the client's own `AGENTS.md`:

```markdown
Generic e-Grip 4 documentation is installed in vendor/grip/egrip4-docs/docs/. Read the relevant
document there before changing the platform setup of this implementation.
```

## Contributing

* Only **generic** information belongs here: nothing that is specific to one client (no client names,
  domains, credentials, database names, module lists of a specific site, etc.).
  Client-specific plans and notes belong in the client repository or in internal repositories.
* This repository is **public**: never add secrets, internal hostnames that are not already public,
  credentials, API keys or personal data.
* Write in English, use Markdown, one topic per file in `docs/`, and add every new document to the table above.
* Releasing: set the new `version` in `composer.json`, move the `NEXT VERSION` notes in `changelog.txt` under
  that version, commit, and push a tag with the same version (`1.0.1` or `v1.0.1`). The
  [Release workflow](.github/workflows/release.yml) checks that the tag matches `composer.json`, builds the package
  zip (`docs/`, `AGENTS.md`, `README.md`, `changelog.txt`, `composer.json`) and uploads it to the Composer channel.
  It uses the organization-wide `COMPOSER_UPLOAD_USER` / `COMPOSER_UPLOAD_PASSWORD` settings.

See [AGENTS.md](AGENTS.md) for the rules that apply to AI coding agents working in this repository.
