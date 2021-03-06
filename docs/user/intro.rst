.. _introduction:

Introduction
============

Shorten is based around a :class:`dict`-like object called a `store`. 
Stores allow insertion, deletion and lookup of data through an interface similar 
to a Python :class:`dict`, the primary difference being that store `keys` 
are generated automatically by a `key generator` (or `keygen`).

The corresponding usage between a Python :class:`dict` and store is illustrated 
below:

==================================       ====================
store                                    dict
==================================       ====================
``key, token = store.insert(val)``       ``dict[key] = val``
``del store[token]``                     ``del dict[key]``
``key in store``                         ``key in dict``
``for key in iter(store)`` [#f1]_        ``for key in iter(dict)``
``store.has_token(token)``               no equivalent
``store.get_token(key)``                 no equivalent
``len(store)`` [#f1]_                    ``len(dict)``
==================================       ====================

.. [#f1] This functionality may be missing from some stores due to
         inefficiency or thread safety.

When data is inserted into a store a :class:`Pair <shorten.Pair>` is returned. Pairs 
are tuples containing `key` and `token` attributes. Keys are used to lookup
data, while tokens are used to delete data. Tokens are generated from
`token generators`, similar to how keys are generated from `key generators`.

Making a Store
---------------

There are three built-in store types:

==============================  ========================
store                           notes
==============================  ========================
:class:`shorten.MemoryStore`    native Python.
:class:`shorten.MemcacheStore`  requires ``memcached`` or Memcached-compatible server.
:class:`shorten.RedisStore`     requires a Redis server.
==============================  ========================

All stores are available at the module level. The only required 3rd-party library is
`redis-py <https://github.com/andymccurdy/redis-py>`_, needed for `WatchError`. This
dependency may be removed at a later date.

::

   from shorten import MemoryStore, MemcacheStore, RedisStore

Redis stores require a Redis client (`redis <https://pypi.python.org/pypi/redis/>`_
is assumed) and the name of a key that the keygen will use as a counter.

::

   import redis
   from shorten import RedisStore

   client = redis.Redis()
   store = RedisStore(redis_client=client, 
      counter_key='your_ns_here:counter_key')


Memcache stores require a Memcached client
(`pylibmc <https://pypi.python.org/pypi/pylibmc/>`_ is recommended, although it
requires the C development libraries to compile). The name of the key that will
be used as a counter is also required.

::

   import pylibmc
   from shorten import MemcacheStore

   client = pylibmc.Client(["127.0.0.1"], binary=True,
                     behaviors={"tcp_nodelay": True, "ketama": True})

   store = MemcacheStore(memcache_client=client, 
      counter_key='your_ns_here:counter_key')


A :class:`MemoryStore <shorten.MemoryStore>` does not have any dependencies, 
so it will be used for the remaining examples.

Inserting Values
----------------

Use :meth:`insert` to insert values. A :class:`Pair <shorten.Pair>` is returned, which can
destructured like a tuple.

::

   from shorten import MemoryStore

   store = MemoryStore()
   key, token = store.insert('aardvark')

   # 'aardvark'
   store[key]

   # True
   key in store

   pair = store.insert('bonobo')

   # Pair(key='0', token='0')
   print(pair)

Revoking Keys
-------------

After insertion, keys can be revoked, which will remove the key and its
value from the store.

::

   from shorten import MemoryStore

   store = MemoryStore()
   key, token = store.insert('aardvark')

   del store[token]

   # False
   key in store

Customizing Key Generation
--------------------------

A store's keys are generated with a 
:class:`KeyGenerator <shorten.BaseKeyGenerator>`. The default key generators 
use a counter to increment keys, then convert that number to a string in 
an `alphabet`.

.. admonition:: Randomized Alphabets

   All the examples return keys that are clearly sequential and so you may
   decide to shuffle the alphabet to produce keys that appear random.
   
   Although they appear random, they're not: the alphabet order can easily be
   reconstructed from frequency counting and 
   `Benford's law <https://en.wikipedia.org/wiki/Benford's_law>`_,
   allowing someone to predict all future keys.

   **Never use short URLs to hide your data** - use UUIDs or authentication
   instead.

Alphabets can be anything that is indexable, as long as each symbol in the
alphabet is not contained within any other symbol.
For instance, ``('00', '0', '1')`` would be an ambiguous alphabet, since ``00`` 
could be interpreted as either the symbol ``00`` or two ``0`` symbols.

Keys of a minimum length or starting at a certain *unencoded* value can be
generated by specifying `min_length` or `start`. 

For example, hex keys can be generated:

::

   from shorten import MemoryStore

   hexabet = '0123456789abcdef'
   store = MemoryStore(alphabet=hexabet, min_length='2')

   # '10'
   # '11'
   # '12'
   for i in range(0, 2):
      pair = store.insert('aardvark')
      print(pair.key)


and more exotic alphabets can be constructed as well:

::

   from shorten import MemoryStore
   
   emoticons = (':)', ':(', ':D', ';)', ';(', 'D:', ':o', ':/')
   emote_store = MemoryStore(alphabet=emoticons, start=12)

   key, token = emote_store.insert('aardvark')

   # ':(:D'
   key


Customizing Token Generation
----------------------------

A token generator can be any object with a ``create_token(key)`` method.
Shorten has provides token generator classes:

*  :class:`UUIDTokenGenerator <shorten.UUIDTokenGenerator>` which returns
   UUID4 (randomized) tokens.
*  :class:`TokenGenerator <shorten.TokenGenerator>` returns the key itself.

See :ref:`token-gen-example` for a more comprehensive example.

Formatters
----------

A :class:`Formatter <shorten.Formatter>` is used to format the internal 
representation of a key or token. This is useful for Redis and SQL databases, 
which often need to prefix keys and columns in order to avoid clashes.

Any class or mixin with methods ``format_token(token)`` and 
``format_key(key)`` can be used.

There are two formatters included in Shorten: 

*  :class:`Formatter <shorten.Formatter>` (the default) and
*  :class:`NamespacedFormatter <shorten.NamespacedFormatter>`, which takes a 
   string ``namespace`` and prefixes keys and tokens with ``'{namespace}:keys'`` 
   and ``'{namespace}:tokens'`` respectively.

A Redis Example
~~~~~~~~~~~~~~~

A :class:`NamespacedFormatter <shorten.NamespacedFormatter>` can keep 
keys and tokens from overwriting each other and unrelated data in Redis:

::

   import redis
   from shorten import RedisStore, NamespacedFormatter

   formatter = NamespacedFormatter('testing')

   store = RedisStore(redis_client=redis, 
      counter_key = 'testing:counter_key',
      formatter=formatter)


If a value is inserted, the stored key will be ``'testing:keys:0'``
in Redis, but the returned key will be ``'0'``

::

   key, token = store.insert('aardvark')

   # '0'
   key

   # True
   redis.exists('testing:keys:0')

