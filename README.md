# e-Grip 4 documentation

Generic documentation for building, maintaining and upgrading e-Grip 4 implementations.
The repository contains Markdown files only and is distributed as a Composer package, so every
client implementation can carry an up-to-date copy of the docs as a (dev) dependency.

## Contents

| Document | Description |
|---|---|
| [docs/upgrade-egrip3-to-egrip4.md](docs/upgrade-egrip3-to-egrip4.md) | Upgrading an e-Grip 3.x client branch to e-Grip 4.0 (also usable as instructions for AI coding agents) |
| [docs/developers-guide.md](docs/developers-guide.md) | **Developer's guide** (index of the chapters below): how the e-Grip 4 framework works |
| [docs/developers-guide/application-framework.md](docs/developers-guide/application-framework.md) | Developer's guide 1: application framework and the `Context` object |
| [docs/developers-guide/plugins-and-events.md](docs/developers-guide/plugins-and-events.md) | Developer's guide 2: `PluginManager` and `EventDispatcher` |
| [docs/developers-guide/routing-and-modules.md](docs/developers-guide/routing-and-modules.md) | Developer's guide 3: `RouteRegistry` and `ModuleRegistry` |
| [docs/developers-guide/entities.md](docs/developers-guide/entities.md) | Developer's guide 4: `EntityFinder` and `EntityManager` |
| [docs/developers-guide/core-services.md](docs/developers-guide/core-services.md) | Developer's guide 5: database, storage, mail, services, filters, key-value stores |
| [docs/developers-guide/cms-engine.md](docs/developers-guide/cms-engine.md) | Developer's guide 6: CMS engine, content types and Twig renderer functions |
| [docs/developers-guide/module-exceptions.md](docs/developers-guide/module-exceptions.md) | Developer's guide 7: module exceptions in a client branch |
| [docs/developers-guide/database-updates.md](docs/developers-guide/database-updates.md) | Developer's guide 8: database updates and the schema manager |
| [docs/developers-guide/command-line-scripts.md](docs/developers-guide/command-line-scripts.md) | Developer's guide 9: command line scripts |
| [docs/developers-guide/working-on-plugins-and-libraries.md](docs/developers-guide/working-on-plugins-and-libraries.md) | Developer's guide 10: working on plugins and Composer libraries |
| [docs/developers-guide/new-site.md](docs/developers-guide/new-site.md) | Developer's guide 11: setting up a new site |

## Example implementation

[GripOnline/egrip-demo](https://github.com/GripOnline/egrip-demo) is a public, minimal e-Grip 4 site. It already has
this package as a `require-dev` dependency, so it is a good place to see how the docs are installed and to try out
what they describe.

## Installing in a client implementation

The package is published on the Grip Composer channel. Add the channel (if the project does not have it yet) and the
package to `composer.json`:

```json
{
    "repositories": [
        { "type": "composer", "url": "https://composer.hosting.grip.nl/" }
    ],
    "require-dev": {
        "grip/egrip-docs": "*"
    }
}
```

or, once the channel is configured, run:

```bash
composer require --dev grip/egrip-docs
```

After `composer update` the docs are available in `vendor/grip/egrip-docs/docs/`. Start with
[docs/developers-guide.md](docs/developers-guide.md) for how the framework works, or with the document for the task
at hand. To get the latest version of the docs later on, run:

```bash
composer update grip/egrip-docs
```

Point people and agents there, for instance from the client's own `AGENTS.md`:

```markdown
Generic e-Grip 4 documentation is installed in vendor/grip/egrip-docs/docs/. Read the relevant
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
