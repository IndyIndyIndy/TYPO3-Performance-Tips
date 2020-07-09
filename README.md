# TYPO3 Performance Tips
A few tips to optimize and improve the performance in TYPO3 websites.

This guide is focusing mostly on TYPO3 and server-side performance. To check out frontend performance in general (independent of the used CMS), I recommend reading the 2019 version of the Frontend Performance Checklist from Smashing Magazine: https://www.smashingmagazine.com/2019/01/front-end-performance-checklist-2019-pdf-pages/

____

## Extension Development
### Caching Extbase Plugins
+ When configuring an Extbase plugin, you can specify the allowed Controller actions and which actions should not be cached at all. (this will internally create a `USER_INT` object in TYPO3):

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
+ Sometimes it can help to deliver a completely cached web page and load the uncached parts of the page via ajax. (The tradeoff would be more http requests in the frontend)
+ If your action really needs to be uncached, but also contains computing-intensive parts, that can be cached, consider using the caching framework to cache and fetch these parts from a CacheBackend. 
+ See: https://docs.typo3.org/typo3cms/CoreApiReference/ApiOverview/CachingFramework/Developer/Index.html?highlight=cache

### Caching Framework
+ By default, many caching tables (like `extbase_datamapfactory_datamap`, `extbase_reflection`, `cf_*`, …) are stored in the default DB connection (usually MariaDB/MySQL) configured in TYPO3. In larger projects with fast growing cache tables, consider switching to a `RedisBackend` with reasonable amounts of memory assigned for Redis.
+ There are several other Caching Backends available (`Typo3DatabaseBackend`, `MemcachedBackend`, `ApcBackend`, and others as well as the possibility to implement your own). 
+ On Linux servers with newer file systems and SSDs, even using a file based CacheBackend could possibly be faster compared to the default `Typo3DatabaseBackend`.
+ The official documentation provides detailled information on how and when to use them and how to properly configure the underlying mechanisms:
https://docs.typo3.org/m/typo3/reference-coreapi/master/en-us/ApiOverview/CachingFramework/FrontendsBackends/Index.html#cache-backends

### $GLOBALS['TSFE']->set_no_cache()
+ Make sure, that your extensions are not using `$GLOBALS['TSFE']->set_no_cache()` everywhere, as this will completely disable the caching. (Often seen in really old extensions during a TYPO3 Upgrade)

____

## Extensions
### Frontend search
+ Simple search solutions like `EXT:indexed_search` are only viable for smaller projects. - For providing a scalable and high-performance search (and to benefit from many other advanced search features) use a search platform like `Elasticsearch` or `Solr`.  

## EXT:staticfilecache
+ For cached pages, that require no user interaction, you can use the `EXT:staticfilecache` to drastically improve the performance. It creates static html files of the pages and advises the server via simple mod_rewrite rules (nginx configurations also exist) to show the static version of the page without ever executing PHP and TYPO3 in the frontend.

## EXT:image_autoresize
+ Use `EXT:image_autoresize` to automatically resize compress image files upon upload via the TYPO3 Backend. This avoids  large, unoptimized image files accidentally ending up in the frontend, in case an editor forgets to optimize their images before uploading.

____

## Extbase ORM & Doctrine
### Eager loading and lazy loading
+ There are two opposing ways for optimizing the fetching of data objects in an ORM. - Lazy Loading and Eager Loading with their pros and cons.
+ Lazy loading is achieved in TYPO3 Extbase by adding the annotation `@TYPO3\CMS\Extbase\Annotation\ORM\Lazy` to the property in your model that should be lazy loaded. This means, that the underlying objects are not automatically fetched, when the parent object is fetched/created by the ORM. This is improves performance, if your object has many children objects and lots of relations, but you do not need to initally access them all the time. Lazy Loading instead adds a “proxy” class as a placeholder that automatically fetches the objects once you try to access them (which menas a new DB request). 
+ Depending on the way you use and access your objects (imagine iterating through all the lazy loaded objects in a for loop), this means switch out a single db request with multiple joins with many individual requests, leading to much slower performance. This is a case, where Eager Loading (fetch all relations at once) would be a better choice. 

### Extbase ORM vs Doctrine Querybuilder
+ The usage of Extbase repositories with its Object Relational Mapping and DDD design principles can be a very convenient and easy way to handle data objects, quickly create code and create a running project in a well-structured way. However, using an ORM, and its need to build all the data objects with all its complex relationships in the background, always means a performance tradeoff. If domain objects are not needed and performance critical data handling is required (like import/exports of large data sets), you should rather use the Doctrine Querybuilder for direct db access or the TYPO3 Core Engine.

### Indices for orderBy fields
+ Look at the `$defaultOrderings` array in your Extbase repositories. These db fields could probably benefit from db indices to optimize db performance (configured in your ext_tables.sql).

### Pagination 
Pagination in TYPO3 / Extbase is usually implemented with a standard LIMIT/OFFSET SQL query. While these are no problem on smaller data sets, if the table to be queried grows substantially large, the further you try to paginate the records with this pattern, the slower the queries will become. Imagine you have a table with 50.000 rows and paginate far into with: 
"*SELECT * FROM my_table ORDER BY date LIMIT 40000, 20*". 

What actually happens is, that the db now has to fetch 40.020 records from the table and then drop the first 40.000 to just get the 20 rows that were required. 
In this case you probably want to use keyset navigation. Instead of paginating with an OFFSET, the rows are fetched by using the last known key (uid) from the previous result set and fetch the next *x* rows after this key. There exist several solutions for keyset pagination and the optimal solution often differs from your used database (because the supported db features differ wildly). For MySQL as an example there exist several solutions, you can look at one here:
http://allyouneedisbackend.com/blog/2017/09/24/the-sql-i-love-part-1-scanning-large-table/

Downside to keyset pagination: It is'nt that easy to deeply navigate into a specific page (for instance "go to page 20"), because you will likely not now from which key you have to start. But for "infinite loading" (like seen in facebook, twitter, etc.) it is the perfect solution and even provides more stable result. (you won't see a previously fetched record again, if a new record happens to appear at the top)

____

## Fluid
### Compilable Viewhelpers
+ Make sure your custom Viewhelpers are using the `CompilableInterface`, if possible. This makes the Viewhelper static, avoids the instantiation of many instances of the viewhelper class (which can easily happen if it is placed inside a <f:for>-loop) and improves the performance of the template parsing.

### cache.disable ViewHelper
+ Beware of the ViewHelper `<f:cache.disable />`. If used, it will disable the caching and compiling of the complete Fluid template (not just the single one where the Viewhelper is used) to PHP classes, slowing down the template building.

____

## Install Tool
### [FE][pageNotFoundOnCHashError]
+ If set to `true` and a page request in the frontend fails due to a cHash comparison fail (the calculated chash does not match the actual get parameters in the url), TYPO3 will trigger a 404 error. This is the recommended setting, as with `false` TYPO3 would show the page instead with no caching. You should rather fix your urls if cHash errors occur on the page.

### [FE][disableNoCacheParameter]
+ If set to `true`, the infamous `&no_cache=1` will do nothing at all, if present in any url (Check your code, that this change doesn't break anything).

____

## Server / Database
### LIKE ‘%FOOBAR’
+ Please, for everything that is good, do not use a statment like `LIKE ‘%FOOBAR’` in your sql queries. This will do a full db scan and not scale at all.

### Opcache
+ Check that Opcache is enabled in the phpinfo module. Some cheap hostings completely disable it and other Bytecode caches and require users to upgrade their hosting agreement to enable them.

## Server Infrastructure
+ For larger projects with many visitors, it could be a good idea to invest in multiple servers with a Load Balancer. (Which also improves the uptime of the website)

### DB Indices
+ Optimize your MySQL/MariaDB indices, **especially** for tables that are expected to handle large amounts of data. The `EXPLAIN` statement can help with analyzing your queries as well as the `Debug panel` in the `TYPO3 Adminpanel`.
A short overview for how to use indices: https://de.slideshare.net/myxplain/mysql-indexing-best-practices-for-mysql

### PHP FPM
+ Consider switching to `PHP FPM` (either on Apache or nginx) to better handle large amounts of requests and for other performance improvements compared to the Apache `mod_php` module. 

### PHP Profiling
+ You can use a PHP profiler like `xdebug` or `blackfire.io` (paid service) to find performance bottlenecks in your PHP code.
+ A short introduction for debugging TYPO3 with xdebug and PHPStorm: https://www.youtube.com/watch?v=VtffB0CG1ok

____

## TYPO3 Backend
### TSConfig: TCEMAIN.clearCacheCmd
+ Usually used to automatically clear caches for configured pages, when editing records on another page in the backend (like news records in a sysfolder).
Can be configured with `all` or `pages` to automatically clear **all** caches for every single page. Use this with care. 

### pages.no_cache
+ Before TYPO3 9, it was possible to disable the cache for a page record with the db field `no_cache` (Even for editors). This can heavily affect the CPU load and performance of the website and should be avoided. 

### Mount points
+ Mount points generally are a bad idea (duplicate content), but, if heavily used, can also lead to a lot of db load due to many recursions. 

### Log Tables.
+ Tables like `sys_log`, `sys_history`, etc can grow very large with time and create a bottleneck in the db. Make sure to create a job that regularly truncates old records from these tables. (like the cleanup-scheduler tasks)	
+ Especially deprecation logs should never be enabled on production systems, as they tend to grow very quickly. Even after fixing all the deprecation warnings in your own extensions, you will usually end up with lots of deprecation warnings from third-party extensions that you cannot simply fix (which often happen because these extensions need to support a wide variety of TYPO3 versions). 
You can disable the deprecation log in TYPO3 9 in your `ext_localconf.php` with the following code:

``` php
$GLOBALS['TYPO3_CONF_VARS']['LOG']['TYPO3']['CMS']['deprecations']['writerConfiguration'][\TYPO3\CMS\Core\Log\LogLevel::NOTICE] = [];
```
____

## TypoScript
### config.no_cache = 1
+ This well-known setting completely disables caching in the frontend. Usage of this option is discouraged, but still, I often see this option applied in older TYPO3 websites. (After the customer complained about the horrible performance of the website)
+ You can hide this option behind TS Conditions like `[applicationContext == "Development"]`, to only disable caching on development instances, but even then this can lead to easily overlooked bugs, when only testing features uncached, that, in production, should be cached.

### config.linkVars 
+ If linkVars are used (a classic example is the L parameter before TYPO3 9), you can and should limit the range of allowed values. Like: `config.linkVars = L(0-3)`. These linkVars are automatically added to each link, if present in the current url, and can potentially flood the cache, if no range limit is present.

### config.cache
+ This option determines the lifetime of a pages cache depending on configured db records and their start/stop time. This can for instance be useful on a news page: If a new news record is about to be shown, you will want the old page cache to automatically invalidate. 

``` php
config.cache.10 = tx_news_domain_model_news:11
```
+ This will apply the cache option to the page with the `id`, which then takes the news records in the sysfolder with the id `11` in consideration for the cache lifetime.
+ More info:
https://docs.typo3.org/m/typo3/reference-typoscript/master/en-us/Setup/Config/Index.html#cache

### config.cache_clearAtMidnight
+ With this setting set to `1`, the cache will always expire at midnight.

### config.cache_period
+ Amount of seconds until a page cache should be invalited. `604800` for instance would keep the page cache for a week.

### config.sendCacheHeaders 
+ If set to `1`, TYPO3 will automatically send several cache headers (`Last-Modified: x`, `Expires: x`, `ETag`, `Cache-Control`, `Pragma: public`) for static pages to the browser. Only if the page is cached (No USER_INT objects) and no be_user or fe_user is logged in! 
+ See: https://docs.typo3.org/m/typo3/reference-typoscript/master/en-us/Setup/Config/Index.html#sendcacheheaders

### Typoscript Conditions 
+ Each single TypoScript Condition means a new cache variant of the page (TYPO3 will create one cache for the page, if the condition is met and one if not). To avoid bloating up the cache, use such conditions with care and remove any unnecessary condition stubs in your TypoScript, that are not in use.

### COA_INT / USER_INT
+ Try to limit your \*_INT objects in TypoScript. These are basically uncached components in your website. When present, TYPO3  first creates a static (cached) version of the page and inserts placeholder comments in the html output that mark all locations, where a \*_INT object is present. After that, each single of these objects is created from scratch (uncached). This considerably slows down page generation compared to a fully cached page.
+ Be especially careful, when using \*_INT objects inside a `FLUIDTEMPLATE`. If you place such an object in the section `page.10.variables`, this would **apply the uncached object for every single page, even if it is not actually used in the Fluid template**, unnecessarily slowing down every single page in your frontend.

____

## Using the Adminpanel
### Summary
The adminpanel can help with analyzing some performance bottlenecks in your website, like finding \*_INT objects or quickly analyzing all DB queries on a page. 

### Install
+ `composer require typo3/cms-adminpanel`
+ Install the extension `adminpanel`.
+ Place `config.admPanel = 1` in your TypoScript Setup.

### Usage
+ An icon at the lower right in the frontend appears, with which you can open the adminpanel.
+ The `Info module` shows you the server-side rendering time of the page in milliseconds and if the current page was cached.
+ The label `Count of USER_INT objects` shows you the amount of \*_INT objects on the page. If at least one is present, the page is not cached. Try to avoid these unless absolutely necessary.
+ In the `Debug module` you can see a list of all DB requests and how much time every single one needed for the current page. This can help you find bottlenecks in the DB and to get information, which table fields could benefit from an index.

### Finding \*_INT objects
+ `composer require christianessl/adminpanel_int`
+ Install the extension `adminpanel_int`.
+ A new tab in the `Info module` will appear which lists you the Typoscript of all \*_INT objects on the page, which makes it easier for you, to search for the source in your code. These can also include uncached Extbase plugins.
+ You can also you the `TypoScript Object Browser` in the `Template` module to search for any occurences (or other keywords like `no_cache`).

