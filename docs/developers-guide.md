# e-Grip 4 developer's guide

This guide describes how e-Grip 4 is set up and how it works. Part of this functionality is also available in a
number of e-Grip 3.x branches. To tell the new functionality apart from the "old" framework, this guide speaks of
the **legacy** framework and the **e-Grip 4** framework.

After a general description of the application framework, the guide explains how the CMS engine works within
that framework, and how custom content types and template functions are implemented. The last chapters cover
day-to-day development tasks.

> **Status:** this guide is a translation of the internal developer's guide. Class, event and package names are
> taken from that guide and have **not all been re-verified** against the current platform code. Where this guide
> and [upgrade-egrip3-to-egrip4.md](upgrade-egrip3-to-egrip4.md) disagree, the upgrade guide (which was checked
> against the code) is leading.

Placeholders: `<site>` (lowercase site name, as in `egrip-<site>`), `<Site>` (PHP namespace segment),
`<MODULE>` (module name such as `CMS`).

## Contents

| # | Chapter | Topics |
|---|---|---|
| 1 | [Application framework](developers-guide/application-framework.md) | `grip/vwm`, the `Context` object, core objects |
| 2 | [Plugins and events](developers-guide/plugins-and-events.md) | `PluginManager`, `plugins.lock`, `EventDispatcher`, event hooks |
| 3 | [Routing and modules](developers-guide/routing-and-modules.md) | `RouteRegistry`, front-end routes, `ModuleRegistry`, `actions.xml` and results |
| 4 | [Entities](developers-guide/entities.md) | `EntityFinder` and `EntityManager` (Doctrine): finding, updating and removing objects |
| 5 | [Core services](developers-guide/core-services.md) | Database connections, storage providers, mail, `ServiceRegistry`, `FilterRegistry`, key-value stores |
| 6 | [CMS engine](developers-guide/cms-engine.md) | `Cms.Engine` service, `ContentTypeRegistry`, Twig templates and custom renderer functions |
| 7 | [Module exceptions](developers-guide/module-exceptions.md) | Overriding module configuration and texts from a client branch |
| 8 | [Database updates](developers-guide/database-updates.md) | `updates.xml`, `SchemaUpdateRegistry`, the schema manager |
| 9 | [Command line scripts](developers-guide/command-line-scripts.md) | `Command` interface, `CommandRegistry`, `script/run.php` |
| 10 | [Working on plugins and libraries](developers-guide/working-on-plugins-and-libraries.md) | Symlinking plugins and Composer libraries from a workspace into a client checkout |
| 11 | [Setting up a new site](developers-guide/new-site.md) | Repository, database and `vwm_config.xml` for a new e-Grip 4 site |
