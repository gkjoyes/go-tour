# Go [maps](https://github.com/golang/go/blob/master/src/runtime/map.go) in action

Maps is a collection of unordered pairs of key-value. It is widely used because it provides fast lookups and values that can retrieve, update or delete with the help of keys.

- [How Maps Are Structured](https://github.com/gkjoyes/go-tour/tree/main/maps#how-maps-are-structured)
- [Bucket Overflow](https://github.com/gkjoyes/go-tour/tree/main/maps#bucket-overflow)
- [How Maps Grow](https://github.com/gkjoyes/go-tour/tree/main/maps#how-maps-grow)
- [Iterating over maps](https://github.com/gkjoyes/go-tour/tree/main/maps#iterating-over-maps)

## How Maps Are Structured

A map is essentially a [hash table](https://en.wikipedia.org/wiki/Hash_table) where data is organized into an array of buckets. The number of buckets always equals a power of two. When a mapping operation occurs, a hash key is generated based on the specified key.

The low-order bits of the hash are used to select a bucket. If we look inside any bucket, we will find two data structures, an array containing the top 8 high-order bits (HOBs) from the same hash key used to select the bucket, which distinguishes each key-value pair stored within the bucket; and a byte array storing the key-value pairs, with the keys and values packed together for the specific bucket.

## Bucket Overflow

There is a reason why keys and values are packed together in a bucket. If they were stored as key/value/key/value, padding allocations would be necessary between each key/value pair to maintain proper alignment boundaries.

A bucket is designed to hold only 8 key-value pairs. If a ninth key needs to be added to a full bucket, an overflow bucket is created and referenced from the respective bucket.

## How Maps Grow

Continuing to add or remove key-value pairs from a map can affect the efficiency of map lookups. The load threshold values that determine when to expand the hash table are based on four factors.

- **% overflow**  : percentage of buckets which have an overflow bucket
- **bytes/entry** : overhead bytes used per key/elem pair
- **hitprobe**    : # of entries to check when looking up a present key
- **missprobe**   : # of entries to check when looking up an absent key

Currently, the code uses the following load threshold values:

| loadFactor | %overflow | bytes/entry | hitprobe | missprobe |
|------------|-----------|-------------|----------|-----------|
| 6.50       | 20.90     | 10.79       |   4.25   | 6.50      |

Expanding the hash table begins by assigning a pointer known as the "old bucket" pointer to the current bucket array. A new bucket array is then allocated to hold twice the number of existing buckets, which may result in large allocations, but since the memory is not initialized, the allocation process is fast.

Once the memory for the new bucket array is allocated, the key-value pairs from the old bucket array can be transferred or "evacuated" to the new array. Evacuations occur when key-value pairs are added or removed from the map. The key-value pairs that were together in an old bucket may be moved to different buckets within the new array, and the evacuation algorithm attempts to distribute the key-value pairs evenly throughout the new bucket array.

This process is delicate, as iterators still need to traverse the old buckets until all of them have been evacuated.

## Iterating over maps

The order in which elements of a map are iterated is not determined and may change from one iteration to the next. If an entry in the map is removed before it is reached during iteration, it will not be included in the iteration. Conversely, if an entry is added to the map during iteration, it may or may not be included in the iteration.

Let’s take a look at what happens when you’re iterating over a map. The function [mapiterinit](https://github.com/golang/go/blob/bd5de19b368536574682c45cca9f7864a4eca6d2/src/runtime/map.go#L816) initiates the iterator, and then the function `mapiternext` is called to retrieve the first element in the map. Here is the portion of the code in `mapiterinit` that calculates the starting point for iteration:

https://github.com/golang/go/blob/457721cd52008146561c80d686ce1bb18285fe99/src/runtime/map.go#L845

We generate a random number with `fastrand()`/`fastrand64()` and use it to determine the starting bucket and a random offset within that bucket. The `mapiternext` function then iterates through the elements, returning the first valid entity, while skipping over any empty ones.

```go
for ; i < bucketCnt; i++ {
    offi := (i + it.offset) & (bucketCnt - 1)
    if isEmpty(b.tophash[offi]) || b.tophash[offi] == evacuatedEmpty {
        // TODO: emptyRest is hard to use here, as we start iterating
        // in the middle of a bucket. It's feasible, just tricky.
        continue
    }
    ...
}
```

The starting element may be empty, meaning the likelihood of retrieving a valid element depends on the number of empty buckets and elements that precede it.

For example, consider a scenario where there is one bucket containing two valid entities, as shown in the example below:

```text
[NULL, NULL, 10, NULL, NULL, NULL, NULL, 20]
```

If we start with elements 0, 1, or 2, we'll get 10. If we start with elements 3, 4, 5, 6, or 7, we'll get 20.

## References

- [Macro View of Map Internals In Go - William Kennedy](https://www.ardanlabs.com/blog/2013/12/macro-view-of-map-internals-in-go.html)
