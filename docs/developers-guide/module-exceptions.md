# Module exceptions

[← Developer's guide](../developers-guide.md)

To override module configuration that comes from plugins in a client branch, create a module directory in the client
branch that only holds the exceptions, and register it as an extra module directory.

## Directory

Example structure for exceptions to the `CMS` module:

```
modules/
  CMS/
    config/
      config.xml
    lang/
      texts.nl.xml
```

The location of the `modules` directory is determined by the site plugin (see below). The internal developer's guide
places it at the top level of the branch, since e-Grip 4 no longer has an `include` directory. Branches upgraded from
e-Grip 3.x keep their exceptions in `include/vwm/modules/<MODULE>/` for the time being; see
[upgrade-egrip3-to-egrip4.md](../upgrade-egrip3-to-egrip4.md) (1.2 and 3.11).

## Registering the directory

Register the `CMS` directory from the example as an extra module directory of the CMS module config provider with
`addModuleDir()`, by listening to the `Legacy.ModuleConfigProvider.initialized` event. Example site plugin:

```php
<?php

namespace Grip\<Site>\Plugin;

use Grip\Vwm\Core\Context;
use Grip\Vwm\Plugin\AbstractPlugin;

class Plugin extends AbstractPlugin
{

    /** @var Context */
    private $context;

    /**
     * {@inheritDoc}
     * @see \Grip\Vwm\Plugin\AbstractPlugin::getEventHooks()
     */
    public function getEventHooks()
    {
        $eventHooks = array();
        $eventHooks['Platform.initialized'] = array($this, 'onPlatformInitialized');
        $eventHooks['Legacy.ModuleConfigProvider.initialized'] = array($this, 'onLegacyModuleConfigProviderInitialized');
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
            if ($moduleConfigProvider->getModuleName() == 'CMS') {
                $moduleDir = $this->context->getAppRoot() . 'modules' . DIRECTORY_SEPARATOR . 'CMS' . DIRECTORY_SEPARATOR;
                $moduleConfigProvider->addModuleDir($moduleDir);
            }
        }
    }
}
```

## Contents of the exception files

`config.xml` and `texts.nl.xml` only need to contain the configuration that is overridden.

* **config.xml**: to change the menu editor, for example, include the complete `<entity name="Menu">`, so it
  overrides the whole entity configuration from the CMS module plugin. The standard entity is in
  `plugins/cms-module/config/config.xml`. Recent `grip/pear-vwm` versions also support merge nodes
  (`mode="merge"`), so only the delta has to be included; see
  [upgrade-egrip3-to-egrip4.md, 1.3](../upgrade-egrip3-to-egrip4.md#13-merge-nodes-in-module-configuration).
* **texts.nl.xml**: only include the texts that are not yet in `texts.nl.xml` of the CMS module plugin.
