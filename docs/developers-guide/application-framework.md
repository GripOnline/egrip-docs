# Application framework

[← Developer's guide](../developers-guide.md)

Most of the e-Grip 4 application framework comes from the `grip/vwm` Composer package. This is the core of the
framework. The package contains interfaces, services and registries that serve the different parts of an
application.

## The Context object

The application framework is accessed through the `Context` object. Only one instance is created per request.
It is passed to the services, so they know in which application context they run and what they should
communicate with. Module- or client-specific services are not offered through standard interfaces; they can
register themselves in the [`ServiceRegistry`](core-services.md#serviceregistry).

The following core objects are available through the `Context`:

* `Config`
* `CacheRegistry`
* `DatabaseConnectionRegistry`
* `EntityManager` / `EntityFinder`
* `EventDispatcher`
* `FilterRegistry`
* `MailService`
* `ModuleRegistry`
* `PluginManager`
* `RouteRegistry`
* `ServiceRegistry`
* `StorageProviderRegistry`
* `KeyValueStoreRegistry`

These objects use **lazy initialization**: they are only initialized the first time they are requested.

The next chapters discuss the core objects roughly in order of importance, starting with
[plugins and events](plugins-and-events.md).
