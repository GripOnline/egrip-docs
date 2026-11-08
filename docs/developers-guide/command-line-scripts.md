# Command line scripts

[← Developer's guide](../developers-guide.md)

For a script that is run from the command line, create a class that implements the `Command` interface and register
it in the `CommandRegistry`. Then run it with:

```bash
php script/run.php <MODULE>:<Command>
```

To register a command, listen to the `CommandRegistry.initialize` event in a plugin, with a listener such as:

```php
/**
 * @param CommandRegistry $commandRegistry
 */
public function onCommandRegistryInitialize($commandRegistry)
{
    $commandRegistry->registerCommand('<MODULE>:Cleanup', 'Grip\\<Site>\\Command\\CleanupCommand');
}
```
