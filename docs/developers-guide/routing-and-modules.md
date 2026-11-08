# Routing and modules

[← Developer's guide](../developers-guide.md)

## RouteRegistry

Every request handled by the application goes through the `RequestDispatcher`. There are two entry points: one for
web requests (`html/index.php`) and one for console requests (`script/run.php`). The request dispatcher finds out
which controller should handle the request and leaves the execution to that controller.

The request dispatcher does not know which controller should handle a request, so it delegates that to the
`RouteRegistry`. When the `RouteRegistry` is initialized, the event `RouteRegistry.initialize` is triggered.
Plugins listening to that event can register their routes in the `RouteRegistry` at that moment. A route consists
of a path and the class of the controller that handles the requests for that route. A path can end in a wildcard,
so requests for underlying URLs match the route as well. The route with the **longest matching path** wins: for a
URL that starts with `/assets/something`, the route `/assets/*` is chosen instead of `/*`. (See
[upgrade-egrip3-to-egrip4.md, 1.1](../upgrade-egrip3-to-egrip4.md#11-routing-and-sites-with-php-templates) for the
exact resolution order, including exact matches and route priorities.)

The route `/*` is normally handled by the `FrontendActionController` from the `grip/cms` package and is registered by
the `egrip-plugin-cms-frontend` plugin. The `FrontendActionController` does what the old `html/index.php` script
used to do: find out which action was requested, with `CMS` / `RequestPage` as the default, and execute that action.
The front-end plugin registers more well-known front-end URLs, among which:

| Path | Controller |
|---|---|
| `/*` | `FrontendActionController` |
| `/assets/*` | `FrontendAssetsController` |
| `/images/cached/*` | `FrontendDynamicImagesController` |
| `/images/uploads/*` | `FrontendUploadsController` |
| `/images/visual.*` | `FrontendVisualController` |
| `/robots.txt` | `FrontendRobotsTxtController` |

## ModuleRegistry

The `ModuleRegistry` holds the configuration of VWM modules, which mainly consists of **actions** and their
configuration. The `ModuleRegistry` is used, and therefore initialized, by an action controller. In practice that is
only the `FrontendActionController` from `grip/cms`, since the back-end (`/vwm/`) still works with the legacy module
registry (`VWM_ModuleRegistry`) from the VWM PEAR package.

To register a module it is enough to point to a module directory, which in most cases is a directory inside a
plugin. The CMS front-end plugin, for example, registers its `modules/CMS` directory as the module directory of the
`CMS` module.

When the action controller requests the `CMS` module from the `ModuleRegistry` (to check whether the requested
action exists in that module), the `ModuleRegistry` checks whether the registered module directory contains
`config/actions.xml`. If it does, the configuration is read from that file. The structure of `actions.xml` closely
resembles the files in legacy e-Grip modules, but it has a few extensions, described below.

### Results

Each action can have several **results**. A result has a name and a type:

* The **name** matches the value returned by the `execute()` method of the action.
* The **type** matches one of the result types configured in the module. The `CMS` module, for example, has the
  result types `render`, `redirect`, `plaintext` and `download`. Its `RequestPage` action has, among others, the
  result `success` of type `render` and the result `error` of type `render`.

A result can be configured with parameters. A result of type `redirect`, for example, takes a parameter `location`
with the redirect target and an optional parameter `status` with `301` or `307`. A configured value can contain a
placeholder (`${name}`); its value is then requested from the action class. The redirect results of `RequestPage`
show how result parameters and placeholders work:

```xml
<action name="RequestPage" class="Grip\Cms\Action\RequestPageAction">
  <result name="success" type="render">
    <param name="renderData">${renderData}</param>
  </result>
  <result name="redirect" type="redirect">
    <param name="location">${redirectLocation}</param>
  </result>
  <result name="redirectPermanent" type="redirect">
    <param name="status">301</param>
    <param name="location">${redirectLocation}</param>
  </result>
</action>
```

Since configuring every possible result for every action is undesirable, a module can define **global results**.
Every action then has those results automatically, and can override them.
