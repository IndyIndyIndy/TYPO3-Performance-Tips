# TYPO3-Performance-Tips
A few tips to optimize and improve the performance in TYPO3 websites

## EXTBASE
___
### Caching Extbase Plugins
+ When configuring an extbase plugin, you can specify the allowed Controller actions as well as which actions should not be cached at all. (which will internally create a `USER_INT` object in TYPO3):

``` php
\TYPO3\CMS\Extbase\Utility\ExtensionUtility::configurePlugin(
   'Cyberhouse.Theme',
   'Pi2',
   [
       'Location' => 'list'
   ],
   // uncached actions:
   [
       'Location' => 'list'
   ]
);
```
+ I often stumble upon plugin configurations, that were initially set up as completely uncached for each action during development and never adapted after that. If your action doesn’t involve server-side user interaction (or something similar), it is probably better off cached.
+ Sometimes it can be useful to deliver a completely cached web page and load uncached parts of the page via ajax. (The trade off would be more http requests in the frontend)
+ If your action needs to be uncached, but also contains computing-intensive parts, that can be cached, consider using the caching framework to cache and fetch these parts from a CacheBackend. See: https://docs.typo3.org/typo3cms/CoreApiReference/ApiOverview/CachingFramework/Developer/Index.html?highlight=cache

### Caching Framework
+ By default, many caching tables (like `extbase_datamapfactory_datamap`, `extbase_reflection`, `cf_*`, …) are stored in the default DB connection (usually MariaDB/MySQL). In larger projects with fast growing cache tables, to switch to a `RedisBackend` with reasonable amounts of memory assigned to redis.
+ There are several other Caching Backends available (`Typo3DatabaseBackend`, `MemcachedBackend`, `ApcBackend`, and others as well as the possibility to implement your own). The official documentation provides detailled information on how and when to use them and how to properly configure the underlying mechanisms:
https://docs.typo3.org/m/typo3/reference-coreapi/master/en-us/ApiOverview/CachingFramework/FrontendsBackends/Index.html#cache-backends
