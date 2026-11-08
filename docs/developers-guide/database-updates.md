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
