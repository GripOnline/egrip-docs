# Database updates

[← Developer's guide](../developers-guide.md)

The so-called schema plugins define which database updates are needed for which version of the plugin. The update file
is registered in the `SchemaUpdateRegistry` by listening to the `SchemaUpdateRegistry.initialize` event (see the site
plugin in [upgrade-egrip3-to-egrip4.md, 3.11](../upgrade-egrip3-to-egrip4.md#311-site-plugin) for an example that
registers `config/updates.xml`).

## File format

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<updates>
  <update id="20210910094321.57adf4f10f4bedc5a7c2" tag="VWM e-Grip 3.3">
    <query>CREATE TABLE IF NOT EXISTS `vwm_bucket` (
      `id` int(8) UNSIGNED NOT NULL AUTO_INCREMENT,
      `guid` varchar(32) NOT NULL DEFAULT '',
      `datetime_created` datetime NOT NULL DEFAULT '0000-00-00 00:00:00',
      `datetime_modified` datetime NOT NULL DEFAULT '0000-00-00 00:00:00',
      PRIMARY KEY (`id`),
      UNIQUE KEY `guid` (`guid`)
      ) ENGINE=InnoDB DEFAULT CHARSET=utf8
    </query>
  </update>
  <update id="20210910094344.0260c113b4bc31c0cefd" tag="VWM e-Grip 3.7" requires="20210910094321.57adf4f10f4bedc5a7c2">
    <query>CREATE TABLE IF NOT EXISTS `vwm_bucketparam` (
      `id` int(8) UNSIGNED NOT NULL AUTO_INCREMENT,
      `bucket_id` int(10) UNSIGNED NOT NULL,
      `name` varchar(50) NOT NULL DEFAULT '',
      `value` text NOT NULL,
      `datetime_created` datetime NOT NULL DEFAULT '0000-00-00 00:00:00',
      `datetime_modified` datetime NOT NULL DEFAULT '0000-00-00 00:00:00',
      PRIMARY KEY (`id`),
      UNIQUE KEY `uniquerecord` (`bucket_id`,`name`)
      ) ENGINE=InnoDB DEFAULT CHARSET=utf8
    </query>
  </update>
</updates>
```

Rules:

* Updates are listed in the right order, so **new updates always go at the bottom**.
* Every update needs a **globally unique** `id`.
* The optional `tag` gives the update a readable description.
* The optional `requires` attribute states that one or more other updates must have been executed first. The example
  refers to the previous update, which is normally not needed since the updates in one file are always in order. It
  is useful to refer to an update from another plugin, for instance an `<update>` with an `INSERT` that refers to the
  update with the `CREATE` statement of the table the record goes into.
* Queries are stored **without** a trailing semicolon.

## Applying updates

Updates are applied to an environment with the schema manager at `/vwm/schemaManager/`. It shows which updates are
still open, and you can execute them with the press of a button. The schema manager shows the queries with a
semicolon, so you can also copy them and execute them by hand one by one, which is advisable for queries that can
take a long time.

Grip has an internal tool that turns semicolon-separated SQL queries into an `<update>` node with one `<query>` node
per query (with the semicolons removed).

### One process per update

Updates can be started by more than one process at the same time, for instance by two administrators, or by the
API during a deploy on several servers that share a database. To prevent an update from being executed twice, the
schema manager claims the updates in the table `vwm_schema_update_lock` first. The table is created by an update of
the VWM schema plugin (`egrip-plugin-vwm-schema` 4.5) and has a unique key on `update_id`:

1. The schema manager inserts a lock record for **every** pending update of the execution plan. If one of the
   inserts fails on the unique key, another process is busy: the schema manager removes the locks it inserted itself
   and executes nothing. The page (or the API) reports which update is locked and by which process.
2. It executes the updates in order. An update that another process executed in the meantime is skipped. After an
   update has been executed and registered in `vwm_schema_update`, its lock record is removed.
3. When a query fails, the schema manager removes all its remaining locks, so the update can be executed again, or
   its remaining queries by hand. The locks are also removed when the process is aborted by a fatal error or a time
   limit (shutdown function).

As long as the lock table does not exist (before the VWM schema update that creates it has been executed), updates
are executed without locks, as before.

A lock can only stay behind when the process or the database server disappears halfway, for instance when the server
is restarted. The schema manager page lists the current locks with the process (host, pid) and the time; remove a
lock that stays behind with `DELETE FROM vwm_schema_update_lock WHERE update_id = '...'` or with the API (below),
but only when the process that holds it no longer runs.

### Schema manager API

With the VWM API plugin (`egrip-plugin-vwm-api`) installed, the schema manager plugin exposes the API
`schemamanager` at `/api/modules/schemamanager/` (documentation at `/api/modules/swagger.html`). It uses the same
authentication as the other APIs: a personal access key or OAuth token (`Authorization: Bearer ...`), or the VWM
session.

| Method | Path | Scope | Description |
|---|---|---|---|
| `GET` | `v1/updates` | `schemaupdates:read` | The pending updates in order of execution, with their queries, missing or circular requirements, whether they can be executed (`executable`) and the current locks |
| `POST` | `v1/updates/execute` | `schemaupdates:write` | Executes all pending updates |
| `DELETE` | `v1/locks/{updateId}` | `schemaupdates:write` | Removes a lock that stayed behind (`204`, or `404` when there is no lock) |

The optional body of `POST v1/updates/execute`, `{"updates": ["<updateId>", ...]}`, holds the pending updates the
client expects, in the order of `GET v1/updates`. When the pending updates differ, nothing is executed and the API
returns `409` with error code `plan_changed`. Without a body, all pending updates are executed. The response holds a
`status`:

* `200`, `success`: all pending updates were executed (`executedUpdates`), `skippedUpdates` lists the updates that
  another process executed in the meantime;
* `409`, `locked`: another process is executing one of the updates (`lockedBy`), nothing was executed;
* `500`, `failed`: a query failed. The updates before it were executed; `failedUpdate`, `failedQuery`,
  `errorMessage` and `remainingQueries` describe the failure, as on the schema manager page.

Missing or circular requirements give `422`. Administrators (group 1) may use both scopes. Other groups need the
scopes in `vwm_group_allowedscope` with module `schemamanager`, and an access key or OAuth client needs the scopes
as well. Example of a deploy step:

```bash
curl -fsS -X POST -H "Authorization: Bearer $EGRIP_ACCESS_KEY" https://www.example.com/api/modules/schemamanager/v1/updates/execute
```
