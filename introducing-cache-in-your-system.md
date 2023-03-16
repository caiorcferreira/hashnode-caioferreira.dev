---
title: "Introducing Cache in your System"
datePublished: Mon May 04 2020 19:00:00 GMT+0000 (Coordinated Universal Time)
cuid: clf8yly4i079mxnnvab4h7w15
slug: introducing-cache-in-your-system
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1678841038565/ecad912c-f5c4-4567-bd00-a68211f16e92.jpeg
tags: software-architecture, caching, devops, distributed-system
---

---

Photo by [Joshua Coleman](https://unsplash.com/@joshstyle?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/storage?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

Caching is one of the most popular tools used to scale systems and anyone looking to maintain high throughput, resilient and cost-effective products should understand how to use it because it is financially impractical to apply only compute resources in order to meet the access demands.

Knowing the basics about it and what parameters you should be looking when choosing your solution is rarely addressed and hence is the purpose of this article.

## Don’t rush your decision

If one would search for caching it will find a plethora of tutorials teaching how to set up your solution. They make it look so easy to use a cache in your application that one may do it without thinking twice. Be careful, every choice comes with costs and tradeoffs, caching is no different.

Usually, we can find bad cache designs when we stumble on the most important metrics for it: hit ratio and miss ratio. As a brief overview, we can define these metrics as follow:

**Hit ratio** : when the cache **has** a key and can provide the value for the system to use, we call it a _hit_. The metric is simply the _number of hits / number of lookups_.

**Miss ratio** : when the cache **hasn’t** the key and the value must be computed for the system to use, we call it a _miss_. The metric is simply the _number of misses / number of lookups_.

The _number of lookups_ is simply the total quantity of cache accesses, which is _lookups = hits + misses_.

You can know that the cache is not being efficient if it has a low hit ratio and, therefore, a high miss ratio. What low or high means will depend on your problem. Discovering your baseline metrics can only be achieved through our first guideline.

## Guidelines

### Monitor your cache

You should set up a way to capture the hit and miss ratio of your cache solution. How you will do it is highly dependable on the chosen implementation, but most of it should have an easy way of extracting these statistics and if not, consider looking for others.

The most important about having monitoring is exactly that you will be able to experiment with different algorithms and tradeoffs presented in the below guidelines and strive to improve your ratios. Hence, formulate a hypothesis and let your data drive your solution.

### Tradeoffs: Performance and Resilience vs Consistency

Caching is one of the simplest and more powerful ideas in computation. Understanding and applying its basic cases is easy but can become extremely hard sooner than you imagine. Therefore the classical phrase, “there is only two difficult things in computer science: cache invalidation and naming things”. But, why cache invalidation is so hard? Because it usually is critical and has many moving parts.

The moving parts come from the fact that caches improve application performance because they bring the data closer, which also means we now have gone off the rails with the most important principle for Consistency: have one source of truth. This has another effect which is resilience, since now if our main source of information goes off, our application can survive a little longer with its cached values.

The critical segment arises usually when you need to update the data on your cache. In order to give your user meaningful information, you need to understand the access patterns to choose the right eviction algorithm and parameters which will balance performance and correctness, choosing the wrong one will probably damage your product. Besides that, there is the case when you need to force clean your cache and the distributed nature of it can cause a lot of pain in the process of invalidating each cache node.

Therefore, adding a cache to your solution is a trade-off between Performance+Resilience versus Consistency and so the first question you should ask yourself is “Can my system live with potentially old and invalid data?” If your answer is Yes, then you can continue here, otherwise, caching will do you more harm than good.

### Keyspace

So you decided that you really want a cache. The first decision you will have to make is about your keyspace, i.e. what you will use as a cache key to index your costly computed values?

It is important because you need to analyze your key cardinality, which means how many distinct values your key can have. For example, a boolean key has a cardinality of 2 (true or false) whereas a customer id can have thousands of possible values. This is really important to understand because a low cardinality key would limit the amount of data your cache could store but your hit ratio would be really high. On the other side, if a key has an extremely high cardinality (tending to uniqueness, never repeating itself) your cache could grow exponentially and you may end up with a low hit ratio, demanding much computation and providing little performance improvements.

Hence you want to choose a high cardinality key, avoiding never-repeating ones, but not to small, avoiding limited ones. One way many uses to achieve this balance is to use complex keys (like a map or list), mixing a medium cardinality key like customer id with a low cardinality one like state names.

One scenario that you should be careful is with memoization. For those coming from OO lands, it is a technique to cache the values computed by a function. It uses the function’s arguments as keys and caches the result. But, since a function may change over time and you may use a solution that wraps the function in a place far from the local here it is implemented, there is a great risk that some feature or refactoring adds unique arguments (like a timestamp) or reduce the arguments to low cardinality ones (a small enum and a boolean). You should implement memoization near to the implementation and/or use cache solutions that allow you to choose the keys from the argument list, which is the best solution since one could not accidentally chance that.

### Cache algorithms and strategies

Next, you need to understand your access pattern in order to choose your cache algorithm and possible strategies. Each one will have trade-offs, you can combine some of them and you should experiment because in this area data will be better to guide you.

- **FIFO (First in First out)**: this is the simplest algorithm where the cache works like a queue and evict the first block to enter, independent of how many times it was used. Through time, you will have the most used keys remaining in the cache, since even if a block was evicted, since it is highly used, duplicated blocks of this key will be presented at the queue. This strategy is really simple to implement and has low overhead but also has an inefficient usage of memory in comparison to other algorithms.
- **LRU (Least Recently Used)**: probably the most used cache algorithm, it tracks when some block was used and evict the one with fewer accesses. Hence it keeps the most used keys in the cache but with better memory usage. As a tradeoff, it has a more complex and costly implementation since it has to add and track age bits in the cache blocks.
- **LFU (Least Frequently Used)**: imagine that you are using an LRU cache and you have 100 accesses in the last second. There were 80 hits in the key A, 19 hits in key B and 1 hit in key C. If your cache is full, in the next miss that needs to load a new key, your cache would not evict C, because it was the most recent one accessed. This can be really bad since we are probably removing a more usage key (A or B) in favor of C. That is the problem that the LFU algorithm addresses since it keep the most frequent usage keys, it would evict the key C from our example because it has a small access frequency. This is a really interesting model and probably is better suited to most use cases, but as a pattern, it also has more overhead than LRU because how it has to keep track of how many times a block was accessed in relation to how many accesses happened to the cache.
- **TTL (Time to Live)**: one really common strategy is time to live, an algorithm that evicts blocks that are older than a certain pre-defined timespan. It is used for more volatile data and usually with two cases: for low cardinality keys that can’t grow the memory footprint to the point where one block would be evicted or in combination with other cache strategies (like the ones mentioned above) to provide more refresh opportunities.
- **Stale data** : we say that a block is stale when it passes its expiration time and should be evicted or refreshed. The stale data strategy is an augmentation of TTL that instead of eliminating the block from the cache once the time to live expires, it runs a refresh function (in case of memoization it reruns the function) that will compute a new value for the key. During this time, the stale (old) data is served for those that access the cache. It can also be the case where if the refresh function fails, it simply maintains the block in the cache, that may be evicted by some algorithm, but is not eliminated by its expiration. This can be a really powerful strategy for increased resilience if your system support living with a possibly long-living stale data.

Besides these there is also more modern algorithms like Windowed TinyLFU (used by [Caffeine](https://github.com/ben-manes/caffeine)), [LIRS](<https://en.wikipedia.org/wiki/Cache_replacement_policies#Low_inter-reference_recency_set_(LIRS)>) and [ARC](<https://en.wikipedia.org/wiki/Cache_replacement_policies#Adaptive_replacement_cache_(ARC)>). Note that various discussions about the cache algorithm will reference the theoretical [Bélady’s algorithm](https://en.wikipedia.org/wiki/Cache_replacement_policies#B%C3%A9l%C3%A1dy's_algorithm), so it is good to have a look at it.

### Local vs Distributed

One last question you might need to answer is if you are going to use a local or distributed cache. This will be the most important question in financial terms, so look close to your needs.

**Local** : it means that you will maintain the data in the memory of an application instance. This is the most simple setup, but it can impose a burden in the allocated RAM, maybe forcing you to upgrade to a bigger instance, which can be pretty expensive. It also can have suboptimal performance-wise, since the worst-case scenario for a cache with a size limit of 1000 keys is to have just it stored across all instances, that is, the same data is replicated in all local caches. This intersection diminishes the total performance gain across your system.

**Distributed** : in this case one would use a solution like Redis or Memcache, running separately from the application, and being shared by all instances. This solution optimizes the resource usage (the RAM is allocated exclusively for the cache), allow for bigger cache entries and improves data density (there is, no more replicated entries), but even then it can be less performant than the local solution since it demands a network roundtrip to access the database. Besides that, the cost to maintain new instances and the skill needed to operate these should not be overlooked.

All these points should be taken into consideration when choosing the design for your cache in your system.

## Conclusion

Therefore, before looking into tutorials and getting started articles about how to set up your cache, think about and discuss your problem and how you could configure the cache to deliver the most value to your system. You can use these and other guidelines as starting points for the debates and focus on guiding your decisions based on the data measured from your implementation.

I hope you found it useful and you can find more of my thoughts at [caioferreira.dev](http://caioferreira.dev/)
