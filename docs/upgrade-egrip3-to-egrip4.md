# Upgrading an e-Grip 3.x client branch to e-Grip 4

This document describes how to upgrade an existing e-Grip 3.x client branch to e-Grip 4 while keeping as much
of the configuration and code as possible in the platform plugins and libraries. It is written for developers
and as working instructions for AI coding agents.

Moving `include/vwm/modules` to `config/modules` and converting PHP templates to Twig are **out of scope**:
both can be done later, after the site runs on e-Grip 4. Until then the PHP templates of the site's style keep working
through `grip/egrip-plugin-legacy-frontend` (1.1).

Placeholders: `<site>` (lowercase site name, e.g. used in `egrip-<site>`), `<Site>` (PHP namespace segment),
`<MODULE>` (module name such as `CMS`), `<Mod>` (module namespace segment such as `Cms`).

---

## 0. Using this document with an AI coding agent

Give the agent access to the client branch (`egrip-<site>`), its style package (`egrip-style-<site>`), the
`egrip3` and `egrip4` reference repositories, and this package. Then use prompts like:

> Read `vendor/grip/egrip-docs/docs/upgrade-egrip3-to-egrip4.md`. Do the **analysis (section 2)** for
> `egrip-<site>` and write the resulting plan to a Markdown file in the client repository (or another internal
> location). Do not modify the client branch yet.

and, after the plan has been reviewed:

> Execute section 3 of the plan on branch `<upgrade-branch>`. Stop at every step marked **[HUMAN]** and tell me
> what I need to do.

Steps marked **[HUMAN]** need a running site, a database, a server or a decision. An agent without those
cannot perform them.

The upgrade is done in **one step**: the client branch goes directly from its 3.x version to e-Grip 4. There is
no intermediate 3.x release. Two one-time scripts, written by the agent for the specific branch, make that possible:

* **`sql/upgrade-4.sql`** brings the database from the state of the 3.x branch to the state the e-Grip 4 schema
  manager expects: it executes the 3.x (and older) schema updates that are still missing and registers all updates
  that are already present in table `vwm_schema_update`. After it has run, the schema manager only shows the
  e-Grip 4 updates as open (2.6, 3.16).
* **`script/upgrade-4-remove-files.sh`** removes the files that the upgrade deleted from the repository from the
  servers as well; unpacking a release only adds and overwrites files (2.7, 3.17).

The agent determines the content of both scripts from the code, the update files of the schema plugins and a
structure dump of the database. Nobody has to execute queries on an empty database or collect queries by hand.

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
* **Sites with PHP templates (old, non-Twig styles) require `grip/egrip-plugin-legacy-frontend`** (≥ 1.0.0).
  On `RouteResolver.initialized` it replaces the controller of the `'/*'` route registered by `cms-frontend`
  (`FrontendActionController`) with `LegacyFrontendActionController`. All other routes keep their e-Grip 4
  controllers:

  ```
  /                       -> LegacyFrontendActionController
  /some/page/             -> LegacyFrontendActionController
  /assets/css/style.css   -> FrontendAssetsController       (more specific route, not affected)
  /vwm/                   -> LegacyBackendActionController
  /robots.txt             -> FrontendRobotsTxtController
  ```

  Without configuration **all** `'/*'` requests go to the legacy controller. Exceptions are configured with
  `frontend_controller` params in `vwm_config.xml` (globally or per `<environment>`), with the value `legacy` or
  `default` (the e-Grip 4 Twig controller). The most specific param wins:
  `frontend_controller[DOMAIN][MODULE][ACTION]`, then module/action params (`frontend_controller[MODULE][ACTION]`,
  `[MODULE][*]`, `[*][ACTION]`), then `frontend_controller[DOMAIN]`, then `frontend_controller`.
  Requests without `module`/`action` count as `CMS` / `RequestPage`. This makes it possible to convert a site to
  Twig gradually, for example per domain:

  ```xml
  <param name="frontend_controller" value="legacy" />
  <param name="frontend_controller[<new-domain>]" value="default" />
  ```

  See the plugin's README for all options. The plugin never switches a route back to the Twig controller, so the
  `ML` / `ViewMailingHTML` exception of `egrip-plugin-vwm-legacy-site` always stays legacy.

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
  installation from the server include_path. The upgraded branch must not go live with the old front controller (see 3.8 and 3.10).
* `actions.xml`, language files and `dataobjects.ini` have no merge mode. Nodes and keys that are identical to
  the standard are removed and the rest is kept as is.
* Review every `mode="merge"` node, and test each affected form or tab (section 4).

### 1.4 dataobjects.ini precedence

`VWM_DataObjectConfigLoader::loadDataObjectsfiles()`: if `include/vwm/modules/<MODULE>/config/dataobjects.ini`
exists it is used **instead of** the schema plugin file (there is no merge). A stale branch file therefore hides
new tables and columns of the plugin. Custom tables of a standard module go into the schema plugin. Custom
modules can define their tables in the DataObject class itself (`table()` / `keys()`).

### 1.5 Composer constraints

* All plugins, including the schema plugins (`grip/egrip-plugin-*-schema`), use `"*"`. The schema plugins are
  not pinned to a 3.x version: their `updates.xml` contains the complete history of the schema (tagged
  `<MODULE> e-Grip <version>`, from 1.0 onwards), which is the input for the one-time database script (2.6).
* Module plugins (`grip/egrip-plugin-*-module`) pull in the `grip/pear-*` libraries they need.
* `grip/pear-shop`, `grip/pear-tcpdf` and `grip/pear-al` are not pulled in by the standard e-Grip 4 set. Require them
  explicitly if the site uses SHOP or PDF output. For SHOP, also require `grip/egrip-plugin-shop-module` and
  `grip/egrip-plugin-shop-schema`.
* `grip/egrip-style-default_clean` (the demo style of the egrip4 project) is **not** required by client branches.
* `grip/egrip-plugin-legacy-frontend` (`"*"`) is required by every site that still has PHP templates (1.1). It is not
  in the egrip4 project's `composer.json`, because that project uses Twig.

### 1.6 The schema manager

Verified against `grip/vwm-schemamanager` 1.0.6:

* Every schema plugin (and the site plugin, 3.13) registers an `updates.xml` on `SchemaUpdateRegistry.initialize`
  (format: [database-updates.md](developers-guide/database-updates.md)).
* An update counts as executed when its `id` is in column `update_id` of table `vwm_schema_update`. Nothing else is
  checked: the schema manager does not look at the actual structure of the database.
* The table is created with:

  ```sql
  CREATE TABLE IF NOT EXISTS `vwm_schema_update` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `update_id` varchar(255) NOT NULL,
    `tag` varchar(255) DEFAULT NULL,
    `datetime_executed` datetime DEFAULT NULL,
    `datetime_created` datetime NOT NULL,
    `datetime_modified` datetime NOT NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `update_id` (`update_id`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8
  ```

* After executing the queries of an update, the schema manager registers it with
  `INSERT INTO vwm_schema_update (update_id, tag, datetime_executed, datetime_created, datetime_modified) VALUES ('<id>', '<tag>', NOW(), NOW(), NOW())`.
  The one-time script (2.6) writes the same rows, as `INSERT IGNORE` so that a row that already exists is left alone.

A 3.x database usually has no `vwm_schema_update` table (3.x updates were applied by hand from `sql/updates.sql`).
Without that table the schema manager would offer **every** update since 1.0 as open, which would fail on
an existing database. That is what the one-time script prevents.

---

## 2. Analysis (no changes to the client branch)

The result of this phase is a plan in the client repository (or another internal location, **not** in this package).
The plan contains everything needed for the upgrade of that specific branch, including the complete one-time
database script and the list of files to remove from the servers.

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

### 2.6 Database: classify every schema update

Inputs:

* The update files of the installed e-Grip 4 schema plugins: `plugins/*-schema/config/updates.xml`. Include
  `shop-schema` when the site uses SHOP.
* The branch version (2.2).
* **[HUMAN]** a structure-only dump of the production database, and the contents of `vwm_schema_update` if that
  table already exists. Structure only: no content and no personal data. Give it to the agent; do not commit it
  to a public repository.

  ```bash
  mysqldump --no-data --skip-triggers <database> > <site>-structure.sql
  ```

  ```bash
  mysqldump --no-create-info <database> vwm_schema_update > <site>-schema-update.sql
  ```

* The branch's own `sql/updates.sql` (the updates that were applied by hand), and the changes in `sql/updates.sql`
  of `egrip3` between the tag that matches the branch version and the last 3.x tag (if that file exists in `egrip3`).

Give every update in the update files one of these actions:

| Action | When | In the one-time script |
|---|---|---|
| `register` | The result of the update is already in the database: the table, column or index exists in the structure dump with the same definition, or the update id is already in `vwm_schema_update`. For an update that only changes data (`INSERT`, `UPDATE`, `DELETE`): its tag version is not higher than the branch version, or the query is in the branch's `sql/updates.sql`. | `INSERT IGNORE INTO vwm_schema_update ...` only |
| `execute` | A 3.x (or older) update whose result is missing in the database. This includes tables of a module the site never used: they are created, so later updates that change them do not fail. | the queries of the update, then the `INSERT IGNORE` |
| `open` | A 4.x update whose result is missing. | nothing; the schema manager executes it after the deploy (3.19) |

Special cases, to be listed in the plan:

* **Partly applied update** (for example one of two columns exists): put the missing queries in the script as
  separate queries, and the update itself as `register`. Mark it for review.
* **A difference in definition** (column type, default, index) between the dump and the update: decide per case.
  Usually `register`, unless the 4.x code depends on the difference. Mark it for review.
* **Queries in `egrip3` `sql/updates.sql` that are not in any `updates.xml`** (not verified whether this occurs):
  add them to the script as separate queries, with a comment, if their result is missing.
* **Site-specific tables** (custom modules, custom tables of standard modules): their existing structure becomes the
  first update of the site's `config/updates.xml` (3.13), with action `register`.

The following script lists the updates and writes the one-time SQL script from the classification. Save it outside the
client repository, e.g. as `/tmp/upgrade-sql.php`. `list` writes one line per update with action `?`. Replace every
`?` with `register`, `execute` or `open` (text after `#` is a comment); `sql` refuses to run while an update is
unclassified or an id is unknown.

```php
<?php
// php upgrade-sql.php list <updates.xml> [<updates.xml> ...] > updates.txt
// php upgrade-sql.php sql updates.txt <updates.xml> [<updates.xml> ...] > sql/upgrade-4.sql
// updates.txt: one line per update, "<id> <register|execute|open> # comment"

function loadUpdates(array $files)
{
    $updates = array();
    foreach ($files as $file) {
        $doc = new DOMDocument();
        if (!$doc->load($file)) {
            fwrite(STDERR, "cannot read $file\n");
            exit(1);
        }
        foreach ($doc->getElementsByTagName('update') as $node) {
            $queries = array();
            foreach ($node->getElementsByTagName('query') as $queryNode) {
                if (trim($queryNode->nodeValue) != '') {
                    $queries[] = trim($queryNode->nodeValue);
                }
            }
            $updates[$node->getAttribute('id')] = array(
                'file' => $file,
                'tag' => $node->getAttribute('tag'),
                'queries' => $queries,
            );
        }
    }
    return $updates;
}

$mode = isset($argv[1]) ? $argv[1] : '';

if ($mode == 'list' && $argc > 2) {
    foreach (loadUpdates(array_slice($argv, 2)) as $id => $update) {
        $firstQuery = preg_replace('/\s+/', ' ', isset($update['queries'][0]) ? $update['queries'][0] : '');
        echo $id . ' ? # ' . $update['tag'] . ' | ' . basename(dirname(dirname($update['file']))) . ' | '
            . substr($firstQuery, 0, 80) . "\n";
    }
    exit(0);
}

if ($mode != 'sql' || $argc < 4) {
    fwrite(STDERR, "usage: php upgrade-sql.php list <updates.xml>...\n       php upgrade-sql.php sql <updates.txt> <updates.xml>...\n");
    exit(1);
}

$updates = loadUpdates(array_slice($argv, 3));
$actions = array();
foreach (file($argv[2]) as $line) {
    $line = trim(preg_replace('/#.*$/', '', $line));
    if ($line != '') {
        list($id, $action) = preg_split('/\s+/', $line);
        $actions[$id] = $action;
    }
}

$errors = array();
foreach ($updates as $id => $update) {
    if (!isset($actions[$id]) || !in_array($actions[$id], array('register', 'execute', 'open'))) {
        $errors[] = "not classified: $id ({$update['tag']})";
    }
}
foreach ($actions as $id => $action) {
    if (!isset($updates[$id])) {
        $errors[] = "unknown update id: $id";
    }
}
if ($errors) {
    fwrite(STDERR, implode("\n", $errors) . "\n");
    exit(1);
}

echo "-- One-time e-Grip 4 upgrade script. Run once, before using the schema manager.\n\n";
echo "CREATE TABLE IF NOT EXISTS `vwm_schema_update` (\n"
    . "  `id` int(11) NOT NULL AUTO_INCREMENT,\n"
    . "  `update_id` varchar(255) NOT NULL,\n"
    . "  `tag` varchar(255) DEFAULT NULL,\n"
    . "  `datetime_executed` datetime DEFAULT NULL,\n"
    . "  `datetime_created` datetime NOT NULL,\n"
    . "  `datetime_modified` datetime NOT NULL,\n"
    . "  PRIMARY KEY (`id`),\n"
    . "  UNIQUE KEY `update_id` (`update_id`)\n"
    . ") ENGINE=InnoDB DEFAULT CHARSET=utf8;\n";

foreach ($updates as $id => $update) {
    if ($actions[$id] == 'open') {
        continue;
    }
    echo "\n-- " . $actions[$id] . ': ' . $update['tag'] . "\n";
    if ($actions[$id] == 'execute') {
        foreach ($update['queries'] as $query) {
            echo $query . ";\n";
        }
    }
    echo "INSERT IGNORE INTO vwm_schema_update (update_id, tag, datetime_executed, datetime_created, datetime_modified) VALUES ('"
        . addslashes($id) . "', '" . addslashes($update['tag']) . "', NOW(), NOW(), NOW());\n";
}
```

```bash
php /tmp/upgrade-sql.php list plugins/*-schema/config/updates.xml > /tmp/updates.txt
```

```bash
php /tmp/upgrade-sql.php sql /tmp/updates.txt plugins/*-schema/config/updates.xml > sql/upgrade-4.sql
```

Pass the same update files in the same order to both commands. Once the site's `config/updates.xml` exists (3.13),
add it at the end of both. Separate queries for the special cases above are added to the generated file by hand, in a clearly
commented block, before the `INSERT IGNORE` of the update they belong to. Keep the classification (`updates.txt`,
with the evidence per update as comment) in the plan.

### 2.7 Files to remove from the servers

Deleting a file in the repository does not delete it on test and production: a release only adds and overwrites
files. Left behind, old module files shadow the standard implementation (1.2, 2.4) and old html scripts bypass the
dispatcher (1.1). The plan therefore contains the list of files and directories to remove, based on the dry run
(2.3 to 2.5): directories that disappear completely (unused modules, module subdirs without exceptions) and the
remaining individual files. The final script is generated from the upgrade branch in 3.17.

Rules for the script:

* It only removes paths that the upgrade branch **no longer contains**. It is therefore safe to run both before and
  after unpacking the release, and it can be shipped in the release.
* Directories are removed as a whole only under `include/`, and only when the upgrade branch has no file left in
  them. Under `html/` it removes individual files only: directories such as `html/images/` and `html/files/` also
  hold uploaded and cached files that are not in the repository.
* **[HUMAN]** (optional) a file list of the application root on production (`find include html script -type f`).
  Files on the server that were never in the repository (left over from earlier manual deploys) are listed in the plan
  as a decision. Under directories that are removed as a whole they disappear anyway.

### 2.8 Decide and write the plan

The plan contains: facts, target `composer.json`, modules to remove, files to delete in the branch (STOCK and html),
code fixes per file, merge nodes to review, the site plugin, the site schema (`config/updates.xml`), the
**classification of every schema update with the one-time script** (2.6), the **files and directories to remove from
the servers** (2.7), open decisions (**[HUMAN]**), risks and a test list. Validate the target composer.json with a
dry run:

```bash
composer update --dry-run --no-interaction --ignore-platform-reqs
```

---

## 3. Execution (on an upgrade branch)

Commit after every step, so each step can be reviewed and reverted on its own.

### 3.1 Upgrade branch
Create the branch from `main` (e.g. `upgrade-4`). Production hotfixes remain possible on `main`; merge them into the
upgrade branch, and check whether they add or remove files (3.17) or database updates (3.16).

### 3.2 composer.json
* Require all plugins from the egrip4 project's `composer.json`, except `grip/egrip-style-default_clean`.
* All plugins `"*"`, including the schema plugins (1.5).
* Add `grip/egrip-plugin-legacy-frontend` when the site's style uses PHP templates (1.1). Without it the
  front-end is rendered by the Twig `FrontendActionController`, which does not know the old style.
* Remove `grip/pear-*` requires that are pulled in anyway. Add the ones that are not (`pear-shop`, `pear-tcpdf`) when the site uses them.
* Add `"psr-4": { "Grip\\": "src/Grip" }` to `autoload` for the site plugin.
* Run `composer update`. Commit `composer.json` only: `composer.lock`, `vendor/`, `plugins/` and `plugins.lock` are git-ignored.

### 3.3 Remove unused modules
Modules such as AL, AW, CRM and SMS are rarely used and do not exist in 4.0. Delete the module dir in
`include/vwm/modules/` and the `<module>` line in `vwm_config.xml`.
**[HUMAN]** confirm in the database that they are unused. Drop their tables after the upgrade is live (3.19).

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

class Plugin extends AbstractPlugin
{

    /** schema name for site specific tables in the schema manager (3.13) */
    const SCHEMA_NAME = '<site>';

    /** @var Context */
    private $context;

    public function getEventHooks()
    {
        $eventHooks = array();

        $eventHooks['Platform.initialized'] = array($this, 'onPlatformInitialized');

        // branch include/vwm/modules/<MODULE>/ dirs act as exceptions on top of the module plugins
        $eventHooks['Legacy.ModuleConfigProvider.initialized'] = array($this, 'onLegacyModuleConfigProviderInitialized');

        // site specific database updates through the schema manager (config/updates.xml)
        $eventHooks['SchemaUpdateRegistry.initialize'] = array($this, 'onSchemaUpdateRegistryInitialize');

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

The site plugin does **not** route the front-end: that is done by `grip/egrip-plugin-legacy-frontend` (1.1).
Do not register your own `'/*'` route for the `LegacyFrontendActionController` as well. A separate route with a
higher prio would always win, and the `frontend_controller` exceptions would no longer work.

### 3.12 vwm_config.xml
Make sure the new framework is activated:

```xml
<param name="context_factory_class" value="Grip\Vwm\Core\DefaultContextFactory"/>
```

Set `version` in `<vwm version="...">` to the e-Grip 4 version of the release.

### 3.13 Site schema updates
From now on all database changes, including those to site-specific tables, go through the schema manager.
Create `config/updates.xml` with the existing state of the site-specific tables as the first update (2.6), and keep
the `SchemaUpdateRegistry.initialize` hook of the site plugin (3.11) enabled. Format:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<updates>
  <update id="<yyyymmddhhmmss>.<unique-hash>" tag="<site> 4.0">
    <query>CREATE TABLE IF NOT EXISTS ...</query>
  </update>
</updates>
```

Use `CREATE TABLE IF NOT EXISTS`, so that the update also works on a new database. The branch's
`sql/updates.sql` is no longer used for new changes; keep it as history.

### 3.14 Build
Make sure the release zip contains `src/` (site plugin) and `config/` next to `html/`, `include/`, `script/` and `sql/`.

### 3.15 Style package
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

### 3.16 One-time database script
Generate `sql/upgrade-4.sql` from the classification in the plan (2.6), with the update files of the plugins as
installed by 3.2 and the site's `config/updates.xml` (3.13). Check that:

* the script starts with the `CREATE TABLE IF NOT EXISTS vwm_schema_update` query,
* every update of every update file is classified, and `sql` ran without errors,
* every `execute` update whose `requires` refers to another `execute` update comes after that update in the script,
* the separate queries for the special cases (2.6) are in the script and commented.

Regenerate the script when the plugins are updated before the release: a new update in a schema plugin is
`open` unless the plan says otherwise.

### 3.17 File removal script
Generate `script/upgrade-4-remove-files.sh` from the difference between `main` and the upgrade branch, after all
deletions have been committed (2.7). Save the generator outside the client repository, e.g. as
`/tmp/make-remove-script.sh`, and run it in the client repository:

```sh
#!/bin/sh
# usage (in the client repository): sh make-remove-script.sh <base> <upgrade-branch> > script/upgrade-4-remove-files.sh
set -eu
BASE=$1
BRANCH=$2
TMP=$(mktemp -d)
git diff --name-only --no-renames --diff-filter=D "$BASE" "$BRANCH" > "$TMP/deleted"
# dirs under include/ in which the upgrade branch has no file left: removed as a whole
while IFS= read -r f; do
    d=$(dirname "$f")
    while [ "$d" != "." ]; do echo "$d"; d=$(dirname "$d"); done
done < "$TMP/deleted" | grep '^include/.' | sort -u | while IFS= read -r d; do
    if [ -z "$(git ls-tree -r --name-only "$BRANCH" -- "$d/")" ]; then echo "$d"; fi
done > "$TMP/dirs"
{
    # topmost of those dirs, then the deleted files that are not inside one of them
    awk '{ for (i in top) if (index($0, top[i] "/") == 1) next; top[NR] = $0; print }' "$TMP/dirs"
    awk 'NR == FNR { top[NR] = $0; next } { for (i in top) if (index($0, top[i] "/") == 1) next; print }' "$TMP/dirs" "$TMP/deleted"
} | sed "s/'/'\\\\''/g; s/^/rm_path '/; s/\$/'/" > "$TMP/body"
cat <<HEAD
#!/bin/sh
# One-time removal of the files deleted by the e-Grip 4 upgrade ($BASE -> $BRANCH).
# Only removes paths that the upgrade branch no longer contains. Use --dry-run first.
# usage: sh script/upgrade-4-remove-files.sh <application root> [--dry-run]
set -eu
cd "\$1"
[ -f vwm_config.xml ] || { echo "\$1 is not an e-Grip application root" >&2; exit 1; }
DRY=\${2:-}
rm_path() {
    [ -e "\$1" ] || return 0
    if [ "\$DRY" = "--dry-run" ]; then echo "remove \$1"; else rm -rf -- "\$1"; fi
}
HEAD
cat "$TMP/body"
rm -rf "$TMP"
```

```bash
sh /tmp/make-remove-script.sh main upgrade-4 > script/upgrade-4-remove-files.sh
```

The generated script contains one `rm_path '<path>'` line per directory or file. Compare it with the list in the plan
and add the decided server-only leftovers (2.7) by hand. Regenerate it after merging hotfixes from `main` (3.1).

### 3.18 Local test [HUMAN]
On a copy of the production database: run `sql/upgrade-4.sql`, open `/vwm/schemaManager/` and check that exactly the
`open` updates of the plan are listed. Apply them there, then test (section 4). If an `execute` query fails, or the
schema manager lists an update that should not be open, fix the classification, regenerate the script and start
again from a fresh copy.

### 3.19 Roll out [HUMAN]
Merge the upgrade branch into `main` and make the e-Grip 4 release. On **test** (with a fresh copy of the
production database) and then on **production** (after a backup of the database and the files):

1. Unpack the release and run `composer update` (or deploy the build that contains `vendor/` and `plugins/`).
2. `sh script/upgrade-4-remove-files.sh <application root> --dry-run`, check the output, then run it without
   `--dry-run`.
3. Run `sql/upgrade-4.sql` once (e.g. with the `mysql` client or phpMyAdmin).
4. Open `/vwm/schemaManager/`: only the `open` updates of the plan are listed. Apply them.
5. Reload PHP-FPM (the database ini and the classloader are cached) and run the test checklist (section 4).
6. **[HUMAN]** update the cron jobs on the server (3.10). Tag the release on `main`.

Drop the tables of removed modules (3.3) in a later release, through `config/updates.xml`. Remove the two one-time
scripts from the branch once production runs on e-Grip 4.

## 4. Test checklist [HUMAN]

* `/vwm/` loads and shows the database update notice; if that notice is missing, the
  `egrip-plugin-vwm-schemamanager` dependency is missing. A white page or out-of-memory error points to a
  `dataobjects.ini` that is not loaded (vwm-schema) or a leftover branch `dataobjects.ini`. The database ini is
  cached, so reload PHP-FPM after fixing.
* Every panel and panel icon (standard and custom) exists and has a title.
* Per module: one standard action, one custom action, and one action with deviating configuration. In particular,
  every entity/tab with `mode="merge"` (1.3) shows the standard fields **plus** the exceptions, in the right order.
* Front-end pages are rendered with the old style, through the `LegacyFrontendActionController`
  (`egrip-plugin-legacy-frontend` installed and in `plugins.lock`), and the `frontend_controller` exceptions, if any, work.
* Front-end: home page, deep pages, every custom front-end action (login, password flows, accounts, forms,
  payments), mailing view in the browser (`?module=ML&action=ViewMailingHTML`), RSS/XML and PDF output, search,
  `/robots.txt`, `/images/cached/...`, `/images/uploads/...`, `/assets/...`, `/files/file.php5?...`, `/mc.php`.
* Console: `php script/run.php -m <MODULE> -a <Action>` for every cron job.
* The PHP error log is clean, including PHP 8.x deprecations.

## 5. Pitfalls

| Pitfall | Effect | Fix |
|---|---|---|
| No `vwm_schema_update` table on a 3.x database | the schema manager offers every update since 1.0; executing them fails | run `sql/upgrade-4.sql` first (2.6, 3.16) |
| Update classified `register` while its result is missing | later code or updates fail on a missing table or column | classify against the structure dump, not only by tag version (2.6) |
| Deleted files left on the server | old module files shadow the standard, old html scripts bypass the dispatcher | `script/upgrade-4-remove-files.sh` (2.7, 3.17) |
| Removing whole directories under `html/` | uploaded and cached files are lost | the removal script only removes whole directories under `include/` (2.7) |
| Cleanup against egrip3 HEAD instead of the matching tag | fewer files recognised as standard | use the matching tag |
| Stale stock copies survive the cleanup | old code shadows the 4.0 standard | `classify-remaining.php` (2.4), delete `STOCK` |
| Case-only differences (`Mail_mime`) | file looks customised | treat as stock |
| Expecting `vwm-legacy-site` to route the front-end | a PHP-template site is rendered by the Twig `FrontendActionController` | require `grip/egrip-plugin-legacy-frontend` |
| Own `'/*'` route in the site plugin next to `egrip-plugin-legacy-frontend` | `frontend_controller` exceptions are ignored | remove the own route |
| Merge nodes read by an old VWM | forms/tabs show only the delta | run on `pear-vwm` from `vendor/` only (3.8, 3.10) |
| Physical legacy scripts in `html/` | bypass the dispatcher, load PEAR from the include_path | delete |
| `require_once 'VWM/...'`, `'DI/...'` in code or style | fatal errors or mixed library versions | remove, rely on classmap autoloading |
| `src/` or `config/` missing from the release zip | site plugin or site schema missing on test/production | add them to the build (3.14) |
| `plugins.lock` not updated for a new plugin | plugin not loaded | add the entry by hand |
| `COMPOSER_ALLOW_SUPERUSER` not set in a root container | installer plugin skipped, empty `plugins/` | set it |
