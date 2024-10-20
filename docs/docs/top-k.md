---
title: Top-K
linkTitle: Top-K
description: Top-K is a probabilistic data structure that allows you to find the most frequent items in a data stream.
type: docs
stack: true
weight: 50
---

Top K is a probabilistic data structure in Redis Stack used to estimate the `K` highest-rank elements from a stream.

"Highest-rank" in this case means "elements with a highest number or score attached to them", where the score can be a count of how many times the element has appeared in the stream - thus making the data structure perfect for finding the elements with the highest frequency in a stream.
One very common application is detecting network anomalies and DDoS attacks where Top K can answer the question: Is there a sudden increase in the flux of requests to the same address or from the same IP?
 
There is, indeed, some overlap with the functionality of Count-Min Sketch, but the two data structures have their differences and should be applied for different use cases. 

The Redis Stack implementation of Top-K is based on the [HeavyKeepers](https://www.usenix.org/conference/atc18/presentation/gong) algorithm presented by Junzhi Gong et al. It discards some older approaches like "count-all" and "admit-all-count-some" in favour of a "**count-with-exponential-decay**" strategy which is biased against mouse (small) flows and has a limited impact on elephant (large) flows. This ensures high accuracy with shorter execution times than previous probabilistic algorithms allowed, while keeping memory utilization to a fraction of what is typically required by a Sorted Set and also having the additional benefit of being able to get real time notifications when elements are added or removed from the Top K list.

## Use cases

**Leader boards (gaming)** 

This application answers this question: Who are the K players with the highest score?

Data flow is the incoming game scores. Flow id is the user id, and the value is the score. A separate sorted set is kept, storing the top K users' ids and their scores. 
Every time a user scores points, they're added to the Top K list. 

- If the result is `nil`, the user is already in the Top K. Update the sorted set with the user's new score. 
- if the result is an id of another player, the player you just added (`id1`) took over the player who got returned (`id2`). Remove player `id2` from the sorted set and add player `id1`. 

**Trending hashtags (social media platforms, news distribution networks)** 

This application answers these questions: 

- What are the K hashtags people have mentioned the most in the last X hours? 
- What are the K news with highest read/view count today? 

Data flow is the incoming social media posts from which you parse out the different hashtags. 

The `TOPK.LIST` command has a time complexity of `O(K)` so if `K` is small, there is no need to keep a separate set or sorted set of all the hashtags. You can query directly from the Top K itself. 

## Examples

* Initialize a Top-K with specific parameters
```
> TOPK.RESERVE my-topk 50 2000 7 0.925
```

* Add elements to the Top-K

Multiple items can be added at once. If an item enters the Top-K list, the item which is expelled is returned. This allows dynamic heavy-hitter detection of items being entered or expelled from Top-K list.* 

```
> TOPK.ADD my-topk foo bar 42
```

* Return list of the top K items

```
> TOPK.LIST my-topk 
```

* Check whether an item is one of Top-K items
```
> TOPK.QUERY my-topk 42
```

## Sizing

Choosing the size for a Top K sketch is relatively easy, because the only two parameters you need to set are a direct function of the number of elements (K) you want to keep in your list.

 ```
 > TOPK.RESERVE key k width depth decay_constant
 ```

If you start by knowing your desired `k` you can easily derive the width and depth:

```
width = k*log(k)
depth =  log(k)  # but a minimum of 5
```

For the `decay_constant` you can use the value `0.9` which has been found as optimal in many cases, but you can experiment with different values and find what works best for your use case.

## Performance
Insertion in a top-k has time complexity of O(K + depth) ≈ O(K) and lookup has time complexity of O(K), where K is the number of top elements to be kept in the list and depth is the number of hash functions used.


## Academic sources
- [HeavyKeeper: An Accurate Algorithm for Finding Top-k Elephant Flows.](https://yangtonghome.github.io/uploads/HeavyKeeper_ToN.pdf)

## References
- [Meet Top-K: an Awesome Probabilistic Addition to RedisBloom](https://redis.com/blog/meet-top-k-awesome-probabilistic-addition-redisbloom/)