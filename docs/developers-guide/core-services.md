# Core services

[← Developer's guide](../developers-guide.md)

## DatabaseConnectionRegistry

The framework talks to the database through PDO. The database is configured the same way as in the legacy
framework, with the `database` config parameter in the same syntax as before. It will rarely be necessary to talk to
the database directly, since objects are usually read and written through the [`EntityManager`](entities.md). When
it is necessary, request a connection from the `DatabaseConnectionRegistry` with `getConnection()`, which returns a
PDO object. Several connections can be configured in one environment, although that is rarely needed.

## StorageProviderRegistry

The framework has its own interface for handling files. These can be local files, but because they sit behind an
interface, a data directory can at some point be replaced by another storage method such as Swift or S3.

Storage providers are configured in `vwm_config.xml`. The `vwm-legacy-site` plugin provides `data/uploads` and
`cache/dynimages` by default, each linked to the corresponding local directory with a `LocalFileStorageProvider`.

Working with a storage provider directly is uncommon in a site. The CMS module already uses them: the front-end uses
the storage providers above for uploads and dynamic images.

## MailService

Use the `MailService` to send mail. Several services for sending mail can be configured in an environment. By
default PHP's `mail()` function is used, but an SMTP service can be configured as well. This is optional and needs
the `vwm-messagesender-smtp` package.

Different services can be configured for different sender addresses. The right service, and therefore the right
SMTP configuration, is selected automatically based on the sender address. Example `vwm_config.xml` configuration:

```xml
<!-- mail accounts -->
<param name="message_sender_from[*@example.com]" value="myaccount"/>
<param name="message_sender_from[*@example.org]" value="otheraccount"/>
<param name="message_sender[myaccount][factory]" value="Grip\Vwm\Mail\Service\SMTPMessageSenderFactory"/>
<param name="message_sender[myaccount][smtp_host]" value="localhost"/>
<param name="message_sender[myaccount][smtp_user]" value="<smtp-user>"/>
<param name="message_sender[myaccount][smtp_pass]" value="<smtp-password>"/>
<param name="message_sender[myaccount][smtp_encryption]" value="tls"/>
<param name="message_sender[otheraccount][factory]" value="Grip\Vwm\Mail\Service\SMTPMessageSenderFactory"/>
<param name="message_sender[otheraccount][smtp_host]" value="smtp.example.org"/>
<param name="message_sender[otheraccount][smtp_user]" value="<smtp-user>"/>
<param name="message_sender[otheraccount][smtp_pass]" value="<smtp-password>"/>
<param name="message_sender[otheraccount][smtp_encryption]" value="tls"/>
```

Never commit real SMTP credentials to a repository that is not meant for them.

To send a mail:

1. Compose the mail as a `MimeMessage` object.
2. Pass it to the `getMessageSender()` method of the `MailService`. Based on the From address of the message, it
   looks for a `MessageSender` configured for that address or its domain. If there is one, it is returned;
   otherwise the default `MessageSender` is returned.
3. Call `sendMessage()` on the `MessageSender` with the `MimeMessage`.

## ServiceRegistry

Besides the standard services there are services that are used by several modules. Since modules cannot address each
other directly, they can use a shared `ServiceRegistry`. The `ml-services` plugin, for example, defines the
`ML/userSubscriptionService`, which gives access to the `UserSubscriptionService` from the `grip/ml` package. The CMS
module uses this service to subscribe an e-mail address to a mailing list when a standard form contains a field of
type "e-mail with opt-in".

The `ServiceRegistry` is also used when a site needs custom handling in forms, or when content types want to use
shared data providers to retrieve their content.

To register a service, listen to the `ServiceRegistry.initialize` event. On the `ServiceRegistry` object passed
with that event, define one or more services with `registerService()` or `registerServiceFactory()`. Both take the
service name as first parameter. The second parameter is:

* for `registerService()`: the service object;
* for `registerServiceFactory()`: a class name or a `ServiceFactory` object.

Registering the class name of a service factory is the most efficient way: an invocation often registers many
services but needs only a few of them, so this wastes the least resources.

When a service is requested with `getService()`, it is created automatically if needed. If a service factory was
registered, the factory is instantiated first, and then `createService()` is called on it with the name of the
requested service. This way a custom implementation can use one shared factory class that creates different service
objects based on the name.

## FilterRegistry

The `FilterRegistry` lets plugins manipulate simple values in a flexible way, for example to deviate from a default
value. Listen to the `FilterRegistry.initialize` event and register a filter on the `FilterRegistry` object with
`addFilter()`. When `doFilter()` is called somewhere, a `FilterChain` is built with all filters registered for that
filter name. The filters are executed one by one with the current value. A filter can leave the value alone or change
it, and it can stop the chain, so it is certain which value remains.

## KeyValueStoreRegistry

One or more `KeyValueStore` services can be registered in an installation, through configuration parameters in
`vwm_config.xml`. A key-value store is a simple database with one or more tables in which elements are stored and
are only retrieved, overwritten or removed by their key. An iterator can be used to iterate over all values in a
table.
