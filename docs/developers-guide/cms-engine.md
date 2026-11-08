# CMS engine

[← Developer's guide](../developers-guide.md)

The CMS engine used by the front-end is available as the `Cms.Engine` service, which the `cms-services` plugin
registers automatically. When the CMS engine is first used, it is initialized by the `EngineServiceFactory` from the
`grip/cms` package.

The resulting `Engine` object contains a number of registries, each of which sends out events when it is initialized.
By listening to those events, plugins can work on the registry, for instance to register custom implementations or
to override a standard implementation.

## ContentTypeRegistry

The most important registry is the `ContentTypeRegistry`. It is partly filled from the database and then extended by
plugins. For each content type, not only a model class is defined (matching a database table), but also a
`ContentLoader`, `MenuDataProvider`, `MenuChildrenProvider`, `ContentDataProvider` and `ContentPrincipal` can be
configured. If one of these is not registered for a content type, the global provider known in the
`ContentTypeRegistry` is used.

* **ContentLoader**: a "flat" content type does not need one and can use the default `ContentLoader`. Some content
  types are complex objects made up of data from several database tables; they need a custom `ContentLoader`.
* **MenuDataProvider**: needed when the menu data of a content item contains non-standard fields. The standard fields
  are defined by the `Content` interface (title, browser title, body, etc.). Most standard content types have their
  own `MenuDataProvider` or an exception in the `DefaultMenuDataProvider`. The same applies to
  **MenuChildrenProvider** and **ContentDataProvider**.
* **ContentPrincipal**: decides whether a user may see a content item. Most content types do not define one and use
  the default `ContentPrincipal`. Only "chapters within a page" have a custom `ContentPrincipal`.

To define a custom content type, listen to the `Cms.Engine.ContentTypeRegistry.initialized` event. It is sent after
the standard logic has run, which makes it the right moment to lay exceptions on top.

You can also listen to the `Cms.Engine.initialized` event, which is sent when the CMS engine is fully initialized.
This event can be used, for example, to register custom form builders, form validators or form handlers.

## Twig templates

Front-end pages are rendered with Twig templates by default. See the [Twig documentation](https://twig.symfony.com/doc/)
for the syntax.

To use custom template functions in a site, define your own renderer that extends the `getRendererFunctions()` method
of the default renderer:

```php
class CustomRenderer extends DefaultRenderer
{
    public function getRendererFunctions()
    {
        $functions = parent::getRendererFunctions();
        $functions['getSomeData'] = array($this, 'getSomeData');
        return $functions;
    }

    public function getSomeData($id)
    {
    }
}
```

To activate the custom renderer, register it in the style configuration by listening to the
`Cms.Style.Package.initialized` event:

```php
public function onStylePackageInitialized($stylePackage)
{
    $myRenderer = 'Grip\\<Site>\\CustomRenderer';
    if (($stylePackage) && ($stylePackage->getConfigs())) {
        foreach ($stylePackage->getConfigs() as $styleConfig) {
            $rendererClass = $styleConfig->getRendererClass();
            if ($rendererClass == 'DefaultRenderer') {
                // change to custom renderer
                $styleConfig->setRendererClass($myRenderer);
            }
        }
    }
}
```
