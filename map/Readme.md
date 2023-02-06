# Macro View of Map

Golang Maps is a collection of unordered pairs of key-value. It is widely used because it provides fast lookups and values that can retrieve, update or delete with the help of keys.

## How Maps Are Structured

The hash table for a Go map is structured as an array of buckets. The number of buckets is always equal to a power of 2. When a map operation is performed, a hash key is generated against the key that is specified.

Once a bucket is selected, the key/value pair needs to be stored, removed or looked up, depending on the type of operation. If we look inside any bucket, we will find two data structures. First, there is an array with the top 8 high order bits (HOB) from the same hash key that was used to select the bucket. This array distinguishes each individual key/value pair stored in the respective bucket. Second, there is an array of bytes that store the key/value pairs. The byte array packs all the keys and then all the values together for the respective bucket.

![Hash Map](images/hash-map.png)
