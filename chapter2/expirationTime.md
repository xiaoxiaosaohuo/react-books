## expirationTime

```
// Max 31 bit integer. The max integer size in V8 for 32-bit systems.
// Math.pow(2, 30) - 1
// 0b111111111111111111111111111111
var maxSigned31BitInt = 1073741823;

var NoWork = 0;
var Never = 1;
var Sync = maxSigned31BitInt;

var UNIT_SIZE = 10;
var MAGIC_NUMBER_OFFSET = maxSigned31BitInt - 1;
```
maxSigned31BitInt，通过注释可以知道这是32位系统V8引擎里最大的整数。据粗略计算这个时间大概是12.427天。

####  expirationTime计算公式

```

// 1 unit of expiration time represents 10ms.
function msToExpirationTime(ms) {
  // Always add an offset so that we don't clash with the magic number for NoWork.
  return MAGIC_NUMBER_OFFSET - (ms / UNIT_SIZE | 0);
}

function expirationTimeToMs(expirationTime) {
  return (MAGIC_NUMBER_OFFSET - expirationTime) * UNIT_SIZE;
}

function ceiling(num, precision) {
  return ((num / precision | 0) + 1) * precision;
}

function computeExpirationBucket(currentTime, expirationInMs, bucketSizeMs) {
  return MAGIC_NUMBER_OFFSET - ceiling(MAGIC_NUMBER_OFFSET - currentTime + expirationInMs / UNIT_SIZE, bucketSizeMs / UNIT_SIZE);
}

var LOW_PRIORITY_EXPIRATION = 5000;
var LOW_PRIORITY_BATCH_SIZE = 250;

function computeAsyncExpiration(currentTime) {
  return computeExpirationBucket(currentTime, LOW_PRIORITY_EXPIRATION, LOW_PRIORITY_BATCH_SIZE);
}

// We intentionally set a higher expiration time for interactive updates in
// dev than in production.
//
// If the main thread is being blocked so long that you hit the expiration,
// it's a problem that could be solved with better scheduling.
//
// People will be more likely to notice this and fix it with the long
// expiration time in development.
//
// In production we opt for better UX at the risk of masking scheduling
// problems, by expiring fast.
var HIGH_PRIORITY_EXPIRATION = 500;
var HIGH_PRIORITY_BATCH_SIZE = 100;

function computeInteractiveExpiration(currentTime) {
  return computeExpirationBucket(currentTime, HIGH_PRIORITY_EXPIRATION, HIGH_PRIORITY_BATCH_SIZE);
}
```


