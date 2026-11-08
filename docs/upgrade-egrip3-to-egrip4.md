# Upgrading an e-Grip 3.x client branch to e-Grip 4.0

This document describes how to upgrade an existing e-Grip 3.x client branch to e-Grip 4.0 while keeping as much
of the configuration and code as possible in the platform plugins and libraries. It is written for developers
and as working instructions for AI coding agents.

Moving `include/vwm/modules` to `config/modules` and converting PHP templates to Twig are **out of scope**:
both can be done later, after the site runs on 4.0.

Placeholders: `<site>` (lowercase site name, e.g. used in `egrip-<site>`), `<Site>` (PHP namespace segment),
`<MODULE>` (module name such as `CMS`), `<Mod>` (module namespace segment such as `Cms`).

---

## 0. Using this document with an AI coding agent

Give the agent access to the client branch (`egrip-<site>`), its style package (`egrip-style-<site>`), the
`egrip3` and `egrip4` reference repositories, and this package. Then use prompts like:

> Read `vendor/grip/egrip4-docs/docs/upgrade-egrip3-to-egrip4.md`. Do **Phase 0 (analysis)** for
> `egrip-<site>` and write the resulting plan to a Markdown file in the client repository (or another internal
> location). Do not modify the client branch yet.

and, after the plan has been reviewed:

> Execute Phase 1 of the plan on branch `<upgrade-branch>`. Stop at every step marked **[HUMAN]** and tell me
> what I need to do.

Steps marked **[HUMAN]** need a running site, a database, a server or a decision. An agent without those
cannot perform them.

The upgrade happens in two phases:

1. **Phase 1:** a stripped-down release that is still 3.x: plugins and dispatcher in place, redundant
   code removed, schema plugins pinned to the 3.x version of the branch. It goes to test and production first.
2. **Phase 2:** the actual 4.0 upgrade: schema plugins unpinned, version 4.0, database updates through the
   schema manager.

---

## 1. Background: what changes and why

| Topic | e-Grip 3.x | e-Grip 4.0 |
|---|---|---|
| Libraries | PEAR packages (`VWM/`, `CMS/`, `DI/` ...) from the **server include_path**, or `grip/pear-*` Composer packages | `grip/pear-*` packages in `vendor/`, **classmap autoloaded only** (no include_path) |
| Front controller | `html/index.php` bootstraps `VWM/VWM.php` directly | `html/index.php` → `RequestDispatcher` → route → controller |
| Module code/config | full copy of every module in `include/vwm/modules/<MODULE>/` | defaults from module plugins (`grip/egrip-plugin-*-module`) and `vendor/grip/vwm-legacy-site/src/Modules/<Mod>/{Action,View}`; the branch dir only holds **exceptions** |
| DB structure (`dataobjects.ini`) | `include/vwm/modules/<MODULE>/config/dataobjects*.ini` | schema plugins (`grip/egrip-plugin-*-schema/config/dataobjects*.ini`) |
| DB updates | `sql/updates.sql`, applied by hand | schema manager (`/vwm/schemaManager/`), `updates.xml` per schema plugin, administration in table `vwm_schema_update` |
| Front-end templates | PHP templates in a style package | Twig (`cms-frontend` plugin, `FrontendActionController`) |

### 1.1 Routing, and sites with PHP templates

* The 4.0 `html/index.php` (also used by the latest egrip3) only does:
  ```php
  $dispatcher = RequestDispatcher::createRequestDispatcher(dirname(__DIR__));
  $dispatcher->dispatch();
  ```
  `html/vwm/index.php` and `html/vwm/popup/index.php` just include it.
* Routes are registered by plugins on `RouteRegistry.initialize`. Relevant ones:
  * `egrip-plugin-cms-frontend`: `'/*'` → `Grip\Cms\Controller\FrontendActionController` (**Twig**, default prio 100), plus
    `/assets/*`, `/images/cached/*`, `/images/uploads/*`, `/images/visual.*`, `/images/datasets/*`, `/robots.txt`,
    `/favicon.ico`, `/q/*`, `/co/li/`, `/expiryservice*`.
  * `egrip-plugin-vwm-legacy-site`: `/vwm/`, `/vwm/popup/`, `/vwm/setup/`, `/vwm/modules/CMS/*`, `/pear/*`,
    `/files/file.php5`, `/files/rtefile.php`, `/mc.php*`, `/oauth/authorize`, `/oauth/token`, console route `-m`.
  * `egrip-plugin-vwm-schemamanager`: `/vwm/schemaManager/`.
* `RouteResolver::resolveRouteByRequestUri()` prefers an exact match, then the longest URL pattern, then the
  route with the **highest prio**.
* `Grip\Vwm\LegacySite\Controller\LegacyFrontendActionController` runs a front-end request the 3.x way
  (`VWM::getInstance()` → `VWM_FrontEndController` → `CMS` / `RequestPage`), so **PHP templates keep working**.
* **The `egrip-plugin-vwm-legacy-site` plugin does not send all front-end requests to that controller.** It
  only switches the `'/*'` route for `?module=ML&action=ViewMailingHTML` (`Plugin::onRouteResolverInitialized()`).
  A site with PHP templates must register `'/*'` → `LegacyFrontendActionController` itself, with a prio higher
  than 100. That is done in the site plugin (section 3.11). With the real `RouteResolver` the result is:

  ```
  /                       -> LegacyFrontendActionController
  /some/page/             -> LegacyFrontendActionController
  /assets/css/style.css   -> FrontendAssetsController       (more specific route still wins)
  /vwm/                   -> LegacyBackendActionController
  /robots.txt             -> FrontendRobotsTxtController
  ```

* Physical files in `html/` bypass the dispatcher: the standard `.htaccess` only rewrites requests for files
  that do not exist. Old entry scripts (`html/assets.php`, `html/images/*.php*`, `html/files/*.php5`,
  `html/robots/*`, ...) must be **removed**, otherwise they keep bootstrapping `VWM/VWM.php` from the include_path.

### 1.2 How module configuration is resolved

* `VWM_ModuleRegistry::getModuleConfig()`: a module registered by a module plugin uses the plugin dir. An
  **unregistered** module (a custom module) falls back to `{vwmdir}modules/<MODULE>/`, so custom modules keep
  working from `include/vwm/modules/<MODULE>/` without extra work.
* The `Legacy.ModuleConfigProvider.initialized` hook in the site plugin adds `include/vwm/modules/<MODULE>/` as
  *extra module dir* for standard modules. The branch files then act as exceptions on top of the plugin.
* Action and view classes in a 3.x branch are **global** classes (`class RequestPageAction extends FrontEndAction`).
  Once a branch file is removed, the standard class is
  `Grip\Vwm\LegacySite\Modules\<Mod>\{Action|View}\<Class>`. Note the casing: `Auth`, `Bb`, `Cms`, `Crm`,
  `Ml`, `Shop`, `Vwm`, `Wst`.
* `VWM_ModuleConfig::load()` only reads the include dir. Replace it with `VWM_ModuleConfig::getConfig()`.

### 1.3 Merge nodes in module configuration

Since `grip/vwm-legacy-site` **1.47.0** the cleanup script (`LegacyBranchCleanupController`, see 2.3) no longer
keeps a deviating `<entity>` of a module `config.xml` as a full copy. Instead it writes only the delta, as merge nodes:

* `<entity ... mode="merge">`: an entity that differs from the standard. Tabs that are identical to the standard
  are removed from it.
* `<tab ... mode="merge">`: a form tab that differs. Only the `<input>` nodes that differ or are new are kept.
* `insertbefore="<name>"` on a new `<tab>` or `<input>`: it is placed before that existing tab/input of the
  standard definition.
* A file that contains merge nodes is switched from the DTD to the XSD:
  `xmlns="http://www.e-grip.nl/vwm/module-config"` with schema location
  `https://pear.grip.nl/DTD/VWM/1_76_0/module_config.xsd`.

Example: a branch that only changed the sender options of the mailing schedule form ends up with:

```xml
<entity name="Schedule" classname="ML_Schedule" mode="merge">
  <title name="Schedule" />
  <tab name="main" type="form" mode="merge">
    <title name="Details" />
    <form>
      <input type="select" name="sender">
        ...
      </input>
    </form>
  </tab>
</entity>
```

Consequences:

* Merge nodes are interpreted by `VWM_EntityConfigProvider`, `VWM_FormConfig` and `VWM_TabsConfig` of a
  recent `grip/pear-vwm` (verified in 1.87.2). An **older VWM that does not know `mode="merge"`
  will presumably read the delta as the complete definition** (for instance a form that only shows one
  field). Therefore the cleaned-up branch must always run on the `pear-vwm` from `vendor/`, never on a PEAR
  installation from the server include_path. Phase 1 must not go live with the old front controller (see 3.8 and 3.10).
* `actions.xml`, language files and `dataobjects.ini` have no merge mode. Nodes and keys that are identical to
  the standard are removed and the rest is kept as is.
* Review every `mode="merge"` node, and test each affected form or tab (section 5).

### 1.4 dataobjects.ini precedence

`VWM_DataObjectConfigLoader::loadDataObjectsfiles()`: if `include/vwm/modules/<MODULE>/config/dataobjects.ini`
exists it is used **instead of** the schema plugin file (there is no merge). A stale branch file therefore hides
new tables and columns of the plugin. Custom tables of a standard module go into the schema plugin. Custom
modules can define their tables in the DataObject class itself (`table()` / `keys()`).

### 1.5 Composer constraints

* `"3.7"` in Composer means **exactly 3.7.0**, not "the latest 3.7". Always use `"3.7.*"` for pinned schema
  plugins. (`egrip-plugin-cms-schema` 3.7.0 contains no `dataobjects.ini` or `updates.xml`.)
* Pinned schema plugin versions for Phase 1, by the version of the client branch:

  | Plugin | 3.7.x / 3.8.x | 3.9.x | 3.10.x |
  |---|---|---|---|
  | `grip/egrip-plugin-bb-schema` | `3.7.*` | `3.7.*` | `3.7.*` |
  | `grip/egrip-plugin-cms-schema` | `3.7.*` | `3.9.*` | `3.10.*` |
  | `grip/egrip-plugin-geo-schema` | `3.7.*` | `3.7.*` | `3.7.*` |
  | `grip/egrip-plugin-ml-schema` | `3.7.*` | `3.7.*` | `3.10.*` |
  | `grip/egrip-plugin-shop-schema` (if SHOP is used) | `3.7.*` | `3.7.*` | `3.7.*` |
  | `grip/egrip-plugin-vwm-schema` | `3.7.*` | `3.7.*` | `3.7.*` |
  | `grip/egrip-plugin-wst-schema` | `3.7.*` | `3.7.*` | `3.7.*` |

* Module plugins (`grip/egrip-plugin-*-module`) always use `"*"`. They pull in the `grip/pear-*` libraries they need.
* `grip/pear-shop`, `grip/pear-tcpdf` and `grip/pear-al` are not pulled in by the standard 4.0 set. Require them
  explicitly if the site uses SHOP or PDF output. For SHOP, also require `grip/egrip-plugin-shop-module` and
  `grip/egrip-plugin-shop-schema`.
* `grip/egrip-style-default_clean` (the demo style of the egrip4 project) is **not** required by client branches.

---

## 2. Phase 0: analysis (no changes to the client branch)

The result of this phase is a plan in the client repository (or another internal location, **not** in this package).

### 2.1 Set up the workspace

Install dependencies in the client branch and in the `egrip3` and `egrip4` reference projects. In containers
running as root, Composer disables plugins unless `COMPOSER_ALLOW_SUPERUSER=1` is set. Without plugins the egrip
installer plugin is skipped and `plugins/` stays empty.

```bash
export COMPOSER_ALLOW_SUPERUSER=1
```

```bash
composer install --no-interaction --ignore-platform-reqs
```

```bash
git -C egrip3 fetch --tags
```

The installer plugin puts packages of type `egrip-plugin` in `plugins/<name>` and type `egrip-style` in
`plugins/styles/<name>`. Libraries go to `vendor/grip/`. The `egrip4` project provides the reference set of 4.0
plugins and `vendor/grip/vwm-legacy-site` (cleanup script, legacy controllers, `Modules\*` classes).

### 2.2 Collect facts

| Fact | Where to find it |
|---|---|
| Branch version | `vwm_config.xml` (`<vwm version="...">`), `changelog.txt`, the commit that updated the branch to its current 3.x version |
| Matching egrip3 tag | `git -C egrip3 tag` (`egrip-3.7`, `egrip-3.8`, `egrip-3.9`, `v3.10`, ...) |
| Enabled modules | `<module name="..." enabled="true"/>` in `vwm_config.xml` |
| Custom modules | dirs in `include/vwm/modules/` that never existed in egrip3 |
| Site code outside modules | `include/<SITE>/`, `include/*.php`, `include/lang/`, `script/`, `html/` |
| Style package | `styleroot` param; PHP templates or Twig; Composer package or deployed by hand |
| composer.json | constraint style (`"3.7"` vs `"3.7.*"`), PEAR packages yes/no |
| Front controller | `html/index.php`: legacy bootstrap or dispatcher |
| Site plugin | `src/`, `plugin[...]` params in `vwm_config.xml` |
| Cron jobs | server crontab entries that call `script/` |

### 2.3 Dry-run the cleanup on a copy

```bash
git -C egrip-<site> archive --prefix=<site>-dry/ HEAD | tar -x -C /tmp/dry
```

```bash
git -C egrip3 worktree add --detach /tmp/dry/egrip3-ref egrip-3.9
```

```bash
php egrip-<site>/vendor/grip/vwm-legacy-site/bin/cleanup-branch.php /tmp/dry/<site>-dry /tmp/dry/egrip3-ref
```

* Use the egrip3 **tag that matches the branch version** (above: 3.9), not egrip3 HEAD. The branch is a copy of
  that version, so more files are recognised as identical. Comparing with HEAD leaves more files, sometimes
  including `dataobjects.ini`.
* The script deletes PHP files identical to the reference, writes merge deltas for module config (1.3),
  removes identical nodes from `actions.xml` and language files, cleans `dataobjects.ini` and removes empty dirs.
* Use `vwm-legacy-site` ≥ 1.47.0, the first version with merge support.

The tools used in this phase are part of `grip/vwm-legacy-site`, which every 3.x and 4.0 project already
has installed. Apart from `cleanup-branch.php` they are read-only:

| Tool | Since | Purpose |
|---|---|---|
| `bin/cleanup-branch.php` | earlier versions | removes code and configuration that is identical to the standard (this step; modifies the branch) |
| `bin/classify-remaining.php` | 1.64.0 | finds stale copies of standard files (2.4) |
| `bin/audit-branch.php` | 1.64.0 | finds what breaks in 4.0 (2.5) |
| `bin/compare-dataobjects-ini.php` | 1.64.0 | checks a `dataobjects.ini` against the schema plugin (2.5) |

From 1.64.0 they are also linked in `vendor/bin/`.

### 2.4 Classify what is left

`cleanup-branch.php` only removes files that are identical to the reference. A file that is an **older,
unmodified** copy of a standard file survives. In 4.0 such a file would shadow the newer standard implementation.
`classify-remaining.php` compares every remaining file in `include/` with all historic egrip3 revisions
(ignoring whitespace and `$Id$` lines). It needs `git` and an egrip3 checkout with tags:

```bash
php vendor/grip/vwm-legacy-site/bin/classify-remaining.php /path/to/egrip3 /tmp/dry/<site>-dry egrip-3.9
```

| Class | Meaning | Action |
|---|---|---|
| `STOCK` | identical (ignoring whitespace and `$Id$`) to an older egrip3 revision: an unmodified stale copy | **delete** |
| `MODIFIED` | real customisation | keep; check `require`/`extends`; consider rewriting as a subclass of the `Grip\Vwm\LegacySite\Modules\...` class |
| `NEW` | custom action/view in a standard module | keep; check `require`/`extends` |
| `CONFIG` | XML/INI delta left by the cleanup (possibly with merge nodes) | review manually |
| `CUSTOMMODULE`, `SITE` | custom module / site code | keep |

Check `MODIFIED` files that differ in only a few lines. Often the difference is trivial: for example,
`Mail_mime` vs `Mail_Mime` differs only in case, and PHP class names are case-insensitive, so such a file is effectively `STOCK`.

### 2.5 Audit

Run the audit on the cleaned copy of the branch. For the style package, which has no `include/` dir, use the
`grep` commands at the end of this section.

```bash
php vendor/grip/vwm-legacy-site/bin/audit-branch.php /tmp/dry/<site>-dry
```

It reports:

1. **Broken requires:** `require_once dirname(__FILE__) . '...'` / `__DIR__` requires whose target no longer exists.
   For a missing `include/vwm/modules/<MODULE>/{actions,views}/<Class>.class.php` it names the replacement class
   `Grip\Vwm\LegacySite\Modules\<Mod>\{Action,View}\<Class>` when that class exists in the installed package.
2. **include_path requires** (PEAR style, `'VWM/...'`, `'DI/...'`, ...): in 4.0 these fail, or load a second copy of a library.
3. **`VWM_ModuleConfig::load()` calls**, to be replaced by `VWM_ModuleConfig::getConfig()`.
4. **Leftover `dataobjects*.ini`** in the config dirs of modules.
5. **Front controller and legacy html scripts:** whether `html/index.php` only uses the dispatcher, and every PHP
   file in `html/` other than `index.php`, `vwm/index.php`, `vwm/popup/index.php` and `vwm/stats/index.php`.
6. **Module config files with `mode="merge"` nodes** (1.3), to be reviewed and tested.

Compare each leftover `dataobjects.ini` with the schema plugin's file (`plugins/<mod>-schema/config/`; the `Auth`
tables are in `vwm-schema`). No output (exit code 0) means the branch file is fully covered and can be deleted.
Otherwise the missing definitions belong in the schema plugin:

```bash
php vendor/grip/vwm-legacy-site/bin/compare-dataobjects-ini.php include/vwm/modules/CMS/config/dataobjects.ini plugins/cms-schema/config/dataobjects.ini
```

The same checks by hand, for code without `include/` such as a style package:

```bash
grep -rnE "(require|include)(_once)?\s*\(?\s*dirname" .
```

```bash
grep -rnE "(require|include)(_once)?\s*\(?\s*['\"][A-Z][A-Za-z_]*/" .
```

```bash
grep -rn "VWM_ModuleConfig::load(" .
```

### 2.6 Decide and write the plan

The plan contains: facts, target `composer.json`, modules to remove, files to delete (STOCK and html), code fixes
per file, merge nodes to review, the site plugin, open decisions (**[HUMAN]**), risks and a test list. Validate the
target composer.json with a dry run:

```bash
composer update --dry-run --no-interaction --ignore-platform-reqs
```

---

## 3. Phase 1: stripped 3.x release (on an upgrade branch)

Commit after every step, so each step can be reviewed and reverted on its own.

### 3.1 Upgrade branch
Create the branch from `main` (e.g. `upgrade-4.0`). Production hotfixes remain possible on `main`.

### 3.2 composer.json
* Require all plugins from the egrip4 project's `composer.json`, except `grip/egrip-style-default_clean`.
* Module plugins `"*"`, schema plugins pinned with `.*` (table in 1.5).
* Remove `grip/pear-*` requires that are pulled in anyway. Add the ones that are not (`pear-shop`, `pear-tcpdf`) when the site uses them.
* Add `"psr-4": { "Grip\\": "src/Grip" }` to `autoload` for the site plugin.
* Run `composer update`. Commit `composer.json` only: `composer.lock`, `vendor/`, `plugins/` and `plugins.lock` are git-ignored.

### 3.3 Remove unused modules
Modules such as AL, AW, CRM and SMS are rarely used and do not exist in 4.0. Delete the module dir in
`include/vwm/modules/` and the `<module>` line in `vwm_config.xml`.
**[HUMAN]** confirm in the database that they are unused. Drop their tables once the stripped 3.x release is live.

### 3.4 Run the cleanup script
Against the matching egrip3 tag (2.3).

### 3.5 Delete stale stock copies
The `STOCK` files from 2.4, plus `MODIFIED` files whose differences are trivial.

### 3.6 Fix broken requires and extends
`audit-branch.php` lists them with the replacement class. Remove the `require_once` and extend the fully-qualified class name instead:

```php
<?php
// before
require_once dirname(__FILE__) . '/DefaultEditView.class.php';
class ScheduleEditView extends DefaultEditView
{

// after
class ScheduleEditView extends \Grip\Vwm\LegacySite\Modules\Ml\View\DefaultEditView
{
```

The same applies to static calls (`EditContentAction::getEditActionParams()` →
`\Grip\Vwm\LegacySite\Modules\Cms\Action\EditContentAction::getEditActionParams()`). Classes that extend another
class that is still in the branch keep their `require_once`. Repeat the audit until no broken requires remain.

### 3.7 `VWM_ModuleConfig::load()` → `VWM_ModuleConfig::getConfig()`
In the branch and in the style package. Otherwise only the configuration from the include dir is loaded, not the one from the module plugin.

### 3.8 Remove include_path requires
Remove `require_once 'VWM/...'`, `'DI/...'`, `'CMS/...'`, `'TCPDF/...'` and the like from the branch, its scripts and the
style package. The classes are classmap-autoloaded from `vendor/`. Mixing a server PEAR copy with the vendor copy
causes "Cannot declare class" errors, or silently runs old code that does not understand merge nodes (1.3).

### 3.9 dataobjects.ini
No `dataobjects*.ini` may remain in the config dirs of standard modules (compare first, see 2.5).
A leftover file is a common cause of a white page in `/vwm/`.

### 3.10 Front controllers and scripts
* Copy `html/index.php`, `html/vwm/index.php`, `html/vwm/popup/index.php`, `html/vwm/stats/index.php` and
  `script/run.php` from egrip3.
* Delete legacy html scripts (2.5). Check that every URL they served is covered by a route (1.1). If one is not,
  add a route in the site plugin or a redirect. Check access logs before deleting scripts that might still be linked.
* Port other scripts in `script/` to actions run with `php script/run.php -m <MODULE> -a <Action>`, or bootstrap
  them via `vendor/autoload.php`. **[HUMAN]** update the cron jobs on the server.

### 3.11 Site plugin
Create `src/Grip/<Site>/Plugin/Plugin.php`:

```php
<?php

namespace Grip\<Site>\Plugin;

use Grip\Vwm\Core\Context;
use Grip\Vwm\Plugin\AbstractPlugin;
use Grip\Vwm\Routing\RouteRegistry;

class Plugin extends AbstractPlugin
{

    /** cms-frontend registers '/*' with the default prio 100; equal patterns resolve to the highest prio */
    const LEGACY_FRONTEND_ROUTE_PRIO = 200;

    /** schema name for site specific tables in the schema manager (Phase 2) */
    const SCHEMA_NAME = '<site>';

    /** @var Context */
    private $context;

    public function getEventHooks()
    {
        $eventHooks = array();

        $eventHooks['Platform.initialized'] = array($this, 'onPlatformInitialized');

        // branch include/vwm/modules/<MODULE>/ dirs act as exceptions on top of the module plugins
        $eventHooks['Legacy.ModuleConfigProvider.initialized'] = array($this, 'onLegacyModuleConfigProviderInitialized');

        // PHP-template sites only: all front-end requests through the LegacyFrontendActionController
        $eventHooks['RouteRegistry.initialize'] = array($this, 'onRouteRegistryInitialize');

        // Phase 2: site specific database updates through the schema manager
        // $eventHooks['SchemaUpdateRegistry.initialize'] = array($this, 'onSchemaUpdateRegistryInitialize');

        return $eventHooks;
    }

    /**
     * @param Context $context
     */
    public function onPlatformInitialized($context)
    {
        $this->context = $context;
    }

    /**
     * @param \VWM_ModuleConfigProvider $moduleConfigProvider
     */
    public function onLegacyModuleConfigProviderInitialized($moduleConfigProvider)
    {
        if ($moduleConfigProvider instanceof \VWM_DefaultModuleConfigProvider) {

            $moduleName = $moduleConfigProvider->getModuleName();
            $moduleDir = $this->context->getAppRoot() . 'include' . DIRECTORY_SEPARATOR . 'vwm'
                . DIRECTORY_SEPARATOR . 'modules' . DIRECTORY_SEPARATOR . $moduleName;

            if (is_dir($moduleDir)) {
                $moduleConfigProvider->addModuleDir($moduleDir . DIRECTORY_SEPARATOR);
            }
        }
    }

    /**
     * @param RouteRegistry $routeRegistry
     */
    public function onRouteRegistryInitialize($routeRegistry)
    {
        $routeRegistry->registerRoute(
            'legacyFrontend',
            '/*',
            'Grip\\Vwm\\LegacySite\\Controller\\LegacyFrontendActionController',
            self::LEGACY_FRONTEND_ROUTE_PRIO
        );
    }

    /**
     * @param \Grip\Vwm\SchemaManager\SchemaUpdateRegistry $schemaUpdateRegistry
     */
    public function onSchemaUpdateRegistryInitialize($schemaUpdateRegistry)
    {
        $updateFile = $this->context->getAppRoot() . 'config' . DIRECTORY_SEPARATOR . 'updates.xml';

        if (is_file($updateFile)) {
            $schemaUpdateRegistry->registerUpdateFile(self::SCHEMA_NAME, $updateFile);
        }
    }
}
```

Register it in `vwm_config.xml`:

```xml
<param name="plugin[<site>][class]" value="Grip\<Site>\Plugin\Plugin" />
```

After `composer update`, check that `plugins.lock` contains the plugin. If it does not, add it by hand:
`"<site>":{"class":"Grip\\<Site>\\Plugin\\Plugin"}`. The Composer classloader can be cached stubbornly; restarting PHP-FPM fixes that.

When the site has been converted to Twig templates later, remove the `RouteRegistry.initialize` hook.

### 3.12 vwm_config.xml
Make sure the new framework is activated:

```xml
<param name="context_factory_class" value="Grip\Vwm\Core\DefaultContextFactory"/>
```

### 3.13 Build
Make sure the release zip contains `src/` (site plugin) next to `html/`, `include/`, `script/` and `sql/`.

### 3.14 Style package
Apply 3.7 and 3.8 to the templates. Decide how it is deployed. The preferred option is a Composer package of type
`egrip-style` named `grip/egrip-style-<site>`, which the installer plugin puts in `plugins/styles/<site>`:

```json
{
    "name": "grip/egrip-style-<site>",
    "type": "egrip-style",
    "repositories": [{ "type": "composer", "url": "https://composer.hosting.grip.nl/" }],
    "require": { "grip/egrip-plugin-installer-plugin": "*" }
}
```

### 3.15 Local test [HUMAN]
See section 5.

### 3.16 Register existing database updates [HUMAN]
Existing databases already contain the structure, so the schema plugin updates must be registered, not executed:

1. Open `/vwm/schemaManager/` and check that the updates of all schema plugins are listed.
2. Point the `database` param in `vwm_config.xml` temporarily to an **empty** database, open
   `/vwm/schemaManager/` and apply all updates there.
3. Dump table `vwm_schema_update` (structure + data) into the project. If the branch already has a historic
   `sql/updates.sql`, use a different name, e.g. `sql/schema-manager-baseline.sql`.
4. Point `database` back to the real database. The schema manager now shows open updates.
5. Apply only the baseline SQL (e.g. with phpMyAdmin). The schema manager must now show no open updates.

### 3.17 Release within 3.x and roll out [HUMAN]
* Merge the upgrade branch and make a last 3.x release.
* **Test:** copy the production database. Remove the `include/` dir before unzipping (many files were deleted).
  Run `composer update`. Apply **only** the baseline SQL, then check that the schema manager shows no open updates.
* **Production:** the same, after a database backup.

## 4. Phase 2: e-Grip 4.0

1. Set all schema plugins to `"*"` in `composer.json` (`bb`, `cms`, `geo`, `ml`, `shop` when used, `vwm`, `wst`).
2. Set `version="4.0..."` in `vwm_config.xml`.
3. From now on all database changes, including those to site-specific tables, go through the schema manager.
   Create `config/updates.xml` for the site tables and enable the `SchemaUpdateRegistry.initialize` hook in the
   site plugin. Register the first update (the existing state) as applied on existing databases. Format:
   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <updates>
     <update id="<yyyymmddhhmmss>.<unique-hash>" tag="<site> 4.0">
       <query>CREATE TABLE ...</query>
     </update>
   </updates>
   ```
4. **[HUMAN]** deploy to test → `composer update` → apply the updates via `/vwm/schemaManager/` → deploy to
   production → apply the updates → tag the release on `main`.

## 5. Test checklist [HUMAN]

* `/vwm/` loads and shows the database update notice; if that notice is missing, the
  `egrip-plugin-vwm-schemamanager` dependency is missing. A white page or out-of-memory error points to a
  `dataobjects.ini` that is not loaded (vwm-schema) or a leftover branch `dataobjects.ini`. The database ini is
  cached, so reload PHP-FPM after fixing.
* Every panel and panel icon (standard and custom) exists and has a title.
* Per module: one standard action, one custom action, and one action with deviating configuration. In particular,
  every entity/tab with `mode="merge"` (1.3) shows the standard fields **plus** the exceptions, in the right order.
* Front-end: home page, deep pages, every custom front-end action (login, password flows, accounts, forms,
  payments), mailing view in the browser (`?module=ML&action=ViewMailingHTML`), RSS/XML and PDF output, search,
  `/robots.txt`, `/images/cached/...`, `/images/uploads/...`, `/assets/...`, `/files/file.php5?...`, `/mc.php`.
* Console: `php script/run.php -m <MODULE> -a <Action>` for every cron job.
* The PHP error log is clean, including PHP 8.x deprecations.

## 6. Pitfalls

| Pitfall | Effect | Fix |
|---|---|---|
| Composer constraint `"3.7"` | installs exactly 3.7.0 (cms-schema 3.7.0 has no `dataobjects.ini`/`updates.xml`) | use `"3.7.*"` |
| Cleanup against egrip3 HEAD instead of the matching tag | fewer files recognised as standard | use the matching tag |
| Stale stock copies survive the cleanup | old code shadows the 4.0 standard | `classify-remaining.php` (2.4), delete `STOCK` |
| Case-only differences (`Mail_mime`) | file looks customised | treat as stock |
| Expecting `vwm-legacy-site` to route the front-end | a PHP-template site is rendered by the Twig `FrontendActionController` | `'/*'` route with prio > 100 in the site plugin |
| Merge nodes read by an old VWM | forms/tabs show only the delta | run on `pear-vwm` from `vendor/` only (3.8, 3.10) |
| Physical legacy scripts in `html/` | bypass the dispatcher, load PEAR from the include_path | delete |
| `require_once 'VWM/...'`, `'DI/...'` in code or style | fatal errors or mixed library versions | remove, rely on classmap autoloading |
| `src/` missing from the release zip | site plugin missing on test/production | add it to the build |
| `plugins.lock` not updated for a new plugin | plugin not loaded | add the entry by hand |
| `COMPOSER_ALLOW_SUPERUSER` not set in a root container | installer plugin skipped, empty `plugins/` | set it |
