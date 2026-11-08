# Entities: EntityFinder and EntityManager

[← Developer's guide](../developers-guide.md)

The framework has its own interface for working with database objects: the `EntityManager`. The default
implementation is based on [Doctrine](https://www.doctrine-project.org/) and needs the
`vwm-entitymanager-doctrine` plugin. When that plugin is active, `getEntityManager()` on the `Context` returns a
`DoctrineEntityManager` by default.

Within the `DoctrineEntityManager`, object mappings are registered by listening to the
`PluginMappingFileLocator.initialize` event, as the various schema plugins do. The plugin only has to register its
`mapping.dcm.xml` file in the file locator. The syntax of a mapping file is documented on the
[Doctrine website](https://www.doctrine-project.org/projects/doctrine-orm/en/3.1/reference/xml-mapping.html).

## EntityFinder or EntityManager?

* With an **`EntityManager`** you can write objects (add, update and remove).
* With an **`EntityFinder`** you can only find them.

In Doctrine every object is *managed* by default: a link remains between an object you retrieve and the Doctrine
entity manager. As a result these objects are not cleaned up by the garbage collector, and a script can quickly use
a lot of memory.

The default `EntityFinder` *detaches* every object retrieved through Doctrine immediately. You can no longer modify
those objects, but they are cleaned up by the garbage collector when possible.

Rule of thumb: **use the `EntityFinder` when you do not need to modify objects, and the `EntityManager` otherwise.**

## Finding objects

A single object is retrieved with `findItem()`. Pass the model class and the parameters the object must match (for
example `id = 123`). Optionally you can specify which fields to retrieve; it depends on the implementation whether an
array with those fields or an object with only those fields is returned.

```php
$entityFinder = $context->getEntityFinder();
$object = $entityFinder->findItem('Grip\Foo\Bar', array('id' => '123'));
```

More than one object is retrieved with `findItems()`. Besides the parameters above, this method supports `orders`,
`maxResults` and `offset`: `orders` sets the sort order, `maxResults` and `offset` set how many records to retrieve
and from which record to start.

```php
$objects = $entityFinder->findItems('Grip\Foo\Bar', array('typeId' => '2'));
```

## Updating objects

To save an object, use an `EntityManager` and call `persist()` followed by `flush()`:

```php
$entityManager = $context->getEntityManager();
$object = $entityManager->findItem('Grip\Foo\Bar', array('id' => '123'));
$object->setTitle('foo');
$entityManager->persist($object);
$entityManager->flush();
```

Note that on `flush()` Doctrine saves **all** objects that were retrieved through the entity manager and modified
since. To prevent an object from being saved on a flush, detach it from the entity manager with `detachEntity()`.
An `EntityFinder` does this automatically.

## Removing objects

To remove an object, call `remove()` followed by `flush()`:

```php
$entityManager->remove($object);
$entityManager->flush();
```

Here too, `flush()` saves all modified entities, as described above.
