# Working on plugins and libraries

[← Developer's guide](../developers-guide.md)

When you change a plugin or one of the Composer libraries that Grip maintains, it is convenient to work in an existing
client checkout, so you can test the change right away. You do not want a git project inside another git project,
though, because then you cannot commit. The solution is to symlink the package installed by Composer to a checkout in
your workspace directory, outside the client checkout.

The examples assume client checkouts in `~/vhosts/<site>` and package checkouts in `~/workspace`.

## Plugins

Plugins are installed in `plugins/`:

```bash
cd ~/vhosts/<site>/plugins
rm -rf cms-module
ln -s ~/workspace/egrip-plugin-cms-module cms-module
```

This assumes a checkout of `egrip-plugin-cms-module` in `~/workspace/egrip-plugin-cms-module`. The name of the symlink
must match the name of the **plugin** (`cms-module`), not the name of the repository.

## Composer libraries

Libraries are installed in `vendor/grip/`:

```bash
cd ~/vhosts/<site>/vendor/grip
rm -rf vwm-legacy-site
ln -s ~/workspace/php-vwm-legacy-site vwm-legacy-site
```

This assumes a checkout of `php-vwm-legacy-site` in `~/workspace/php-vwm-legacy-site`. The name of the symlink must
match the name of the **library** (`vwm-legacy-site`), not the name of the repository.

A `composer update` in the client checkout replaces the symlink with the released package again.
