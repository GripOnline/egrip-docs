# Plugins and events

[← Developer's guide](../developers-guide.md)

## PluginManager

e-Grip 4 is built around plugins. Plugins are activated in an environment by the `PluginManager`. A plugin can
hook into the application framework in two ways.

1. **A Composer package of type `egrip-plugin`.** These packages always need the
   `egrip-plugin-installer-plugin` to be installed. The installer registers the plugin automatically.
2. **A plugin class in `vwm_config.xml`.** A name and a class are configured together:

   ```xml
   <!-- plugins -->
   <param name="plugin[<site>][class]" value="Grip\<Site>\Plugin\Plugin"/>
   ```

   The class must be loadable by the autoloader and must implement the interface `Grip\Vwm\Plugin\Plugin`. The
   easiest way is to extend `AbstractPlugin`.

The plugins registered in an environment are recorded automatically in the file `plugins.lock` in the root of the
project. This is a generated file, so **never add it to version control**.

## EventDispatcher

The main task of a plugin class is to define **event hooks** in the method `getEventHooks()`. This method is called
on every plugin when the event dispatcher is initialized.

Since the event hooks of all plugins are registered on every request, this must not slow down handling the request
too much. It is therefore good practice to initially only define listeners that in turn listen to the
initialization of other registries. This is how the CMS front-end plugin does it:

```php
public function getEventHooks()
{
    $eventHooks = array();
    $eventHooks['RouteRegistry.initialize'] = 'onRouteRegistryInitialize';
    $eventHooks['ModuleRegistry.initialize'] = 'onModuleRegistryInitialize';
    return $eventHooks;
}
```

All the CMS front-end plugin does here is listen to the initialization of the `RouteRegistry` and the
`ModuleRegistry`, core elements that are discussed in [Routing and modules](routing-and-modules.md).

The value of a hook can be a string or an array:

* **array**: a PHP callback of two elements, an object (or class name) and a method name;
* **string** (as in the example above): the method with that name on the plugin object itself.

An event is triggered with the `postEvent()` method of the `EventDispatcher`, with at least the event name and
usually an array of parameters. Those parameters are passed to the listeners as separate arguments. For example,
the event fired when the `RouteRegistry` is initialized passes the `RouteRegistry` object as its only parameter.
Listeners can therefore run their logic (registering routes) directly on the `RouteRegistry`.
