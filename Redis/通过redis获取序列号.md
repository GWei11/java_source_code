```java
RedisAtomicInteger entityIdCounter = new RedisAtomicInteger(key, redisTemplate.getConnectionFactory());
Integer increment = entityIdCounter.getAndIncrement();
entityIdCounter.expire(liveTime, TimeUnit.DAYS);
```