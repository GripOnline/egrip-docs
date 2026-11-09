# Instructions for AI coding agents

This repository holds **generic** e-Grip 4 documentation in Markdown. It is public and is distributed as
the Composer package `grip/egrip-docs`.

## Rules for changing this repository

1. **Generic only.** Never add client-specific content: no client or site names, client repository names,
   domains, environment names, database names or contents, file lists from a specific client branch, or
   decisions taken for one client. When a document needs an example, use placeholders such as `<site>`,
   `<Site>`, `<MODULE>`, `egrip-<site>`.
2. **Public repository.** Never add secrets, passwords, DSNs, API keys, tokens, personal data or
   e-mail addresses, and no internal infrastructure details beyond what is already public.
3. **Markdown only.** Apart from the package files (`composer.json`, `changelog.txt`, the release workflow), the
   repository only contains Markdown. Documents live in `docs/`, one topic per file, written in English. Code that readers
   need (templates, small scripts, commands) goes in fenced code blocks inside the documents.
4. Keep [README.md](README.md) up to date: every document in `docs/` is listed in its contents table.
   Add a line for every change under `NEXT VERSION` in `changelog.txt`.
5. State facts you verified against the platform code (package names, class names, event names,
   versions) precisely, and mark anything that is not verified as such.

## Using these docs from a client implementation

When this package is installed in a client implementation, the docs are in
`vendor/grip/egrip-docs/docs/`. Read the document that matches the task (for example
`upgrade-egrip3-to-egrip4.md` for a 3.x → 4 upgrade) before making changes, and follow it.
Write client-specific findings and plans in the client repository, never in this package.
