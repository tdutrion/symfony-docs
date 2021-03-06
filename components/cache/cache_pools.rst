.. index::
   single: Cache Pool
   single: APC Cache, APCu Cache
   single: Doctrine Cache
   single: Redis Cache
   single: PDO Cache, Doctrine DBAL Cache

Cache Pools and Supported Adapters
==================================

Cache Pools are the logical repositories of cache items. They perform all the
common operations on items, such as saving them or looking for them. Cache pools
are independent from the actual cache implementation. Therefore, applications
can keep using the same cache pool even if the underlying cache mechanism
changes from a file system based cache to a Redis or database based cache.

Creating Cache Pools
--------------------

Cache Pools are created through the **cache adapters**, which are classes that
implement :class:`Symfony\\Component\\Cache\\Adapter\\AdapterInterface`. This
component provides several adapters ready to use in your applications.

Array Cache Adapter
~~~~~~~~~~~~~~~~~~~

This adapter is only useful for testing purposes because contents are stored in
memory and not persisted in any way. Besides, some features explained later are
not available, such as the deferred saves::

    use Symfony\Component\Cache\Adapter\ArrayAdapter;

    $cache = new ArrayAdapter(
        // in seconds; applied to cache items that don't define their own lifetime
        // 0 means to store the cache items indefinitely (i.e. until the current PHP process finishes)
        $defaultLifetime = 0,
        // if ``true``, the values saved in the cache are serialized before storing them
        $storeSerialized = true
    );

Filesystem Cache Adapter
~~~~~~~~~~~~~~~~~~~~~~~~

This adapter is useful when you want to improve the application performance but
can't install tools like APCu or Redis in the server. This adapter stores the
contents as regular files in a set of directories on the local file system::

    use Symfony\Component\Cache\Adapter\FilesystemAdapter;

    $cache = new FilesystemAdapter(
        // the subdirectory of the main cache directory where cache items are stored
        $namespace = '',
        // in seconds; applied to cache items that don't define their own lifetime
        // 0 means to store the cache items indefinitely (i.e. until the files are deleted)
        $defaultLifetime = 0,
        // the main cache directory (the application needs read-write permissions on it)
        // if none is specified, a directory is created inside the system temporary directory
        $directory = null
    );

Php Files Cache Adapter
~~~~~~~~~~~~~~~~~~~~~~~

This adapter is very similar to the Filesystem adapter, except that the saving creates
a ``.php`` file, which is included on fetch (allowing the file to be saved in OPcache)::

    use Symfony\Component\Cache\Adapter\PhpFilesAdapter;

    $cache = new PhpFilesAdapter(
        // the subdirectory of the main cache directory where cache items are stored
        $namespace = '',
        // in seconds; applied to cache items that don't define their own lifetime
        // 0 means to store the cache items indefinitely (i.e. until the files are deleted)
        $defaultLifetime = 0,
        // the main cache directory (the application needs read-write permissions on it)
        // if none is specified, a directory is created inside the system temporary directory
        $directory = null
    );

APCu Cache Adapter
~~~~~~~~~~~~~~~~~~

This adapter can increase the application performance very significantly,
because contents are cached in the shared memory of your server, which is much
faster than the file system. It requires to have installed and enabled the PHP
APCu extension. It's not recommended to use it when performing lots of write and
delete operations because it produces fragmentation in the APCu memory that can
degrade performance significantly::

    use Symfony\Component\Cache\Adapter\ApcuAdapter;

    $cache = new ApcuAdapter(
        // the string prefixed to the keys of the items stored in this cache
        $namespace = '',
        // in seconds; applied to cache items that don't define their own lifetime
        // 0 means to store the cache items indefinitely (i.e. until the APC memory is deleted)
        $defaultLifetime = 0,
        // if present, this string is added to the namespace to simplify the
        // invalidation of the entire cache (e.g. when deploying the application)
        $version = null
    );

Redis Cache Adapter
~~~~~~~~~~~~~~~~~~~

This adapter stores the contents in the memory of a Redis server. Unlike the APCu
adapter, it's not limited to the shared memory of the current server, so you can
store contents in a cluster of servers if needed.

Before you start, make sure you have Redis running and have created a connection
that implements the ``\Redis``, ``\RedisArray``, ``\RedisCluster`` or ``\Predis``
classes::

    use Symfony\Component\Cache\Adapter\RedisAdapter;

    $cache = new RedisAdapter(
        // the object that stores a valid connection to your Redis system
        \Redis $redisConnection,
        // the string prefixed to the keys of the items stored in this cache
        $namespace = '',
        // in seconds; applied to cache items that don't define their own lifetime
        // 0 means to store the cache items indefinitely (i.e. until the Redis memory is deleted)
        $defaultLifetime = 0
    );

The :method:`Symfony\\Component\\Cache\\Adapter\\RedisAdapter::createConnection`
helper allows creating a connection to a Redis server using a DSN configuration::

    $redisConnection = RedisAdapter::createConnection('redis://localhost');

    $cache = new RedisAdapter($redisConnection);

See the method's docblock for more options.

Memcached Cache Adapter
~~~~~~~~~~~~~~~~~~~~~~~

This adapter stores the contents into a set of `Memcached`_ servers.

Before you start, make sure you have Memcached running and have created a
:phpclass:`Memcached` object::

    use Symfony\Component\Cache\Adapter\MemcachedAdapter;

    $cache = new MemcachedAdapter(
        // the object that stores a valid connection to your Memcached servers
        \Memcached $client,
        // the string prefixed to the keys of the items stored in this cache
        $namespace = '',
        // in seconds; applied to cache items that don't define their own lifetime
        // 0 means to store the cache items indefinitely (i.e. until the Memcached memory is deleted)
        $defaultLifetime = 0
    );

The :method:`Symfony\\Component\\Cache\\Adapter\\Memcached::createConnection`
helper allows creating a connection to a pool of Memcached server using a DSN configuration::

    $memcachedClient = MemcachedAdapter::createConnection(
        'memcached://user:pass@localhost?weight=33',

        // options, including username, password and Memcached::OPT_* options
        array('persistent_id' => '_products_cache')
    );

    // alternate syntax: array notations for the servers
    $memcachedClient = MemcachedAdapter::createConnection(array(
        array('192.168.1.100', 11211, 33),
        array('192.168.1.101', 11211, 33)
    ));

    $cache = new MemcachedAdapter($memcachedClient);

See the method's docblock for more options.

PDO & Doctrine DBAL Cache Adapter
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. versionadded:: 3.2
    The PDO & Doctrine DBAL adapter was introduced in Symfony 3.2.

This adapter stores the cached items a SQL database accessed through a PDO or a
Doctrine DBAL connection::

    use Symfony\Component\Cache\Adapter\PdoAdapter;

    $cache = new PdoAdapter(
        // a PDO, a Doctrine DBAL connection or DSN for lazy connecting through PDO
        $databaseConnectionOrDSN,
        // the string prefixed to the keys of the items stored in this cache
        $namespace = '',
        // in seconds; applied to cache items that don't define their own lifetime
        // 0 means to store the cache items indefinitely (i.e. until the database is cleared)
        $defaultLifetime = 0,
        // an array of options for configuring the database connection
        $options = array()
    );

Chain Cache Adapter
~~~~~~~~~~~~~~~~~~~

This adapter allows to combine any number of the previous adapters. Cache items
are fetched from the first adapter which contains them. Besides, cache items are
saved in all the given adapters, so this is a simple way of creating a cache
replication::

    use Symfony\Component\Cache\Adapter\ApcuAdapter;
    use Symfony\Component\Cache\Adapter\ChainAdapter;
    use Symfony\Component\Cache\Adapter\FilesystemAdapter;

    $apcCache = new ApcuAdapter();
    $fileCache = new FilesystemAdapter();

    $cache = new ChainAdapter(array($apcCache, $fileCache));

When an item is not found in the first adapters but is found in the next ones,
the ``ChainAdapter`` ensures that the fetched item is saved in all the adapters
where it was missing. Since it's not possible to know the expiry date and time
of a cache item, the second optional argument of ``ChainAdapter`` is the default
lifetime applied to those cache items (by default it's ``0``).

Traceable Adapter
~~~~~~~~~~~~~~~~~

This adapter wraps another adapter, and allows you to read a report of all of the
"calls" made to the adapter, including how long actions took, cache hits, misses
and more::

    use Symfony\Component\Cache\Adapter\TraceableAdapter;
    use Symfony\Component\Cache\Adapter\FilesystemAdapter;

    $fileCache = new FilesystemAdapter();

    $cache = new TraceableAdapter($fileCache);
    
    // work with the $cache adapter like normal

    // returns an array of TraceableAdapterEvent describing the calls
    $events = $cache->getCalls();

Proxy Cache Adapter
~~~~~~~~~~~~~~~~~~~

This adapter is useful to integrate in your application cache pools not created
with the Symfony Cache component. As long as those cache pools implement the
``CacheItemPoolInterface`` interface, this adapter allows you to get items from
that external cache and save them in the Symfony cache of your application::

    use Symfony\Component\Cache\Adapter\ProxyAdapter;

    // ... create $nonSymfonyCache somehow
    $cache = new ProxyAdapter($nonSymfonyCache);

The adapter accepts two additional optional arguments: the namespace (``''`` by
default) and the default lifetime (``0`` by default).

Doctrine Cache Adapter
~~~~~~~~~~~~~~~~~~~~~~

This adapter wraps any `Doctrine Cache`_ provider so you can use them in your
application as if they were Symfony Cache adapters::

    use Doctrine\Common\Cache\SQLite3Cache;
    use Symfony\Component\Cache\Adapter\DoctrineAdapter;

    $sqliteDatabase = new \SQLite3(__DIR__.'/cache/data.sqlite');
    $doctrineCache = new SQLite3Cache($sqliteDatabase, 'tableName');
    $symfonyCache = new DoctrineAdapter($doctrineCache);

This adapter also defines two optional arguments called  ``namespace`` (default:
``''``) and ``defaultLifetime`` (default: ``0``) and adapts them to make them
work in the underlying Doctrine cache.

Php Array Cache Adapter
~~~~~~~~~~~~~~~~~~~~~~~

This adapter is a highly performant way to cache static data (e.g. application configuration)
that is optimized and preloaded into OPcache memory storage::

    use Symfony\Component\Cache\Adapter\PhpArrayAdapter;
    use Symfony\Component\Cache\Adapter\PhpFilesAdapter;

    // somehow, decide it's time to warm up the cache!
    if ($needsWarmup) {
        // some static values
        $values = array(
            'stats.num_products' => 4711,
            'stats.num_users' => 1356,
        );

        $cache = new PhpArrayAdapter(
            // single file where values are cached
            __DIR__ . '/somefile.cache',
            // a backup adapter, if you set values after warmup
            new FilesystemAdapter()
        );
        $cache->warmUp($values);
    }

    // ... then, use the cache!
    $cacheItem = $cache->getItem('stats.num_users');
    echo $cacheItem->get();

.. note::

    This adapter requires PHP 7.x and should be used with the php.ini setting
    ``opcache.enable`` on.

Looking for Cache Items
-----------------------

Cache Pools define three methods to look for cache items. The most common method
is ``getItem($key)``, which returns the cache item identified by the given key::

    use Symfony\Component\Cache\Adapter\FilesystemAdapter;

    $cache = new FilesystemAdapter('app.cache');
    $latestNews = $cache->getItem('latest_news');

If no item is defined for the given key, the method doesn't return a ``null``
value but an empty object which implements the :class:`Symfony\\Component\\Cache\\CacheItem`
class.

If you need to fetch several cache items simultaneously, use instead the
``getItems(array($key1, $key2, ...))`` method::

    // ...
    $stocks = $cache->getItems(array('AAPL', 'FB', 'GOOGL', 'MSFT'));

Again, if any of the keys doesn't represent a valid cache item, you won't get
a ``null`` value but an empty ``CacheItem`` object.

The last method related to fetching cache items is ``hasItem($key)``, which
returns ``true`` if there is a cache item identified by the given key::

    // ...
    $hasBadges = $cache->hasItem('user_'.$userId.'_badges');

Saving Cache Items
------------------

The most common method to save cache items is
:method:`Psr\\Cache\\CacheItemPoolInterface::save`, which stores the
item in the cache immediately (it returns ``true`` if the item was saved or
``false`` if some error occurred)::

    // ...
    $userFriends = $cache->get('user_'.$userId.'_friends');
    $userFriends->set($user->getFriends());
    $isSaved = $cache->save($userFriends);

Sometimes you may prefer to not save the objects immediately in order to
increase the application performance. In those cases, use the
:method:`Psr\\Cache\\CacheItemPoolInterface::saveDeferred` method to mark cache
items as "ready to be persisted" and then call to
:method:`Psr\\Cache\\CacheItemPoolInterface::commit` method when you are ready
to persist them all::

    // ...
    $isQueued = $cache->saveDeferred($userFriends);
    // ...
    $isQueued = $cache->saveDeferred($userPreferences);
    // ...
    $isQueued = $cache->saveDeferred($userRecentProducts);
    // ...
    $isSaved = $cache->commit();

The ``saveDeferred()`` method returns ``true`` when the cache item has been
successfully added to the "persist queue" and ``false`` otherwise. The ``commit()``
method returns ``true`` when all the pending items are successfully saved or
``false`` otherwise.

Removing Cache Items
--------------------

Cache Pools include methods to delete a cache item, some of them or all of them.
The most common is :method:`Psr\\Cache\\CacheItemPoolInterface::deleteItem`,
which deletes the cache item identified by the given key (it returns ``true``
when the item is successfully deleted or doesn't exist and ``false`` otherwise)::

    // ...
    $isDeleted = $cache->deleteItem('user_'.$userId);

Use the :method:`Psr\\Cache\\CacheItemPoolInterface::deleteItems` method to
delete several cache items simultaneously (it returns ``true`` only if all the
items have been deleted, even when any or some of them don't exist)::

    // ...
    $areDeleted = $cache->deleteItems(array('category1', 'category2'));

Finally, to remove all the cache items stored in the pool, use the
:method:`Psr\\Cache\\CacheItemPoolInterface::clear` method (which returns ``true``
when all items are successfully deleted)::

    // ...
    $cacheIsEmpty = $cache->clear();

.. _`Doctrine Cache`: https://github.com/doctrine/cache
.. _`Memcached`: http://php.net/manual/en/book.memcached.php
