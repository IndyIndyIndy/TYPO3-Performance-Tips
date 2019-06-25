# TYPO3-Performance-Tips
A few tips to optimize and improve the performance in TYPO3 websites

## Extension Development
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

### $GLOBALS['TSFE']->set_no_cache()
+ Make sure, that your extensions are not using `$GLOBALS['TSFE']->set_no_cache()` everywhere, as this will completely disable the caching. (often seen in really old extensions during a TYPO3 Upgrade)


## Extbase ORM & Doctrine
### Eager loading and lazy loading
+ There are two opposing ways for optimizing the fetching of data objects. - Lazy loading and eager loading with their pros and cons.
+ Lazy loading is achieved in TYPO3 Extbase by adding the annotation `@TYPO3\CMS\Extbase\Annotation\ORM\Lazy` to a property in your model. This means, that the underlying objects are not automatically fetched, when the parent object is created by the ORM. This is improves performance, if your object has many children objects, but you do not need to initally access them. Lazy loading instead adds a “proxy” class as a placeholder that automatically fetches the objects once you try to access them (a new DB request). Depending on the way you use and access your objects, this means switch out a single db request with multiple joins with many individual requests, leading to much slower performance. This is a case, where Eager loading (fetch all relations at once) would be a better choice. 

## Fluid
### Compilable Viewhelpers
+ Make sure your custom viewhelpers are using the `CompilableInterface`. This makes the viewhelper static, avoids the instantiation of many instances of the viewhelper class (which can easily happening if it is placed inside a <f:for>-loop for instance), improving the performance of the template parsing.

### Cache.Disable ViewHelper
+ Beware of the ViewHelper `<f:cache.disable />`. If used, it will disable the caching and compiling of the complete fluid template (not just the single one where the viewhelper is used) to PHP classes, slowing down the template building.



## Install Tool


## Server / Database


## TYPO3 Backend


## TypoScript
