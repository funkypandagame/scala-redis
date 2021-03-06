# Redis Scala client

## Key features of the library

- Native Scala types Set and List responses.
- Transparent serialization
- Connection pooling
- Consistent Hashing on the client.
- Support for basic Redis sharding. http://redis.io/topics/partitioning
- Support for Redis Sentinel

## Information about redis

Redis is a key-value database. It is similar to memcached but the dataset is not volatile, and values can be strings, exactly like in memcached, but also lists and sets with atomic operations to push/pop elements.

http://redis.io

## Requirements

- maven

        <dependency>
			<groupId>org.redis</groupId>
			<artifactId>scala-redis_${scala.version}</artifactId>
			<version>${scalaredis.version}</version>
		</dependency>

## Usage

Start your redis instance (usually redis-server will do it)

And you are ready to start issuing commands to the server(s)

Redis 2 implements a new protocol for binary safe commands and replies

Let us connect and get a key:

    scala> import com.redis._
    import com.redis._

    scala> val r = new RedisClient("localhost", 6379)
    r: com.redis.RedisClient = localhost:6379

    scala> r.set("key", "some value")
    res3: Boolean = true

    scala> r.get("key")
    res4: Option[String] = Some(some value)

Let us try out some List operations:

    scala> r.lpush("list-1", "foo")
    res0: Option[Int] = Some(1)

    scala> r.rpush("list-1", "bar")
    res1: Option[Int] = Some(2)

    scala> r.llen("list-1")
    res2: Option[Int] = Some(2)

Let us look at some serialization stuff:

    scala> import serialization._
    import serialization._

    scala> import Parse.Implicits._
    import Parse.Implicits._

    scala> r.hmset("hash", Map("field1" -> "1", "field2" -> 2))
    res0: Boolean = true

    scala> r.hmget[String,String]("hash", "field1", "field2")
    res1: Option[Map[String,String]] = Some(Map(field1 -> 1, field2 -> 2))

    scala> r.hmget[String,Int]("hash", "field1", "field2")
    res2: Option[Map[String,Int]] = Some(Map(field1 -> 1, field2 -> 2))

    scala> val x = "debasish".getBytes("UTF-8")
    x: Array[Byte] = Array(100, 101, 98, 97, 115, 105, 115, 104)

    scala> r.set("key", x)
    res3: Boolean = true

    scala> import Parse.Implicits.parseByteArray
    import Parse.Implicits.parseByteArray

    scala> val s = r.get[Array[Byte]]("key")
    s: Option[Array[Byte]] = Some([B@6e8d02)

    scala> new String(s.get)
    res4: java.lang.String = debasish

    scala> r.get[Array[Byte]]("keey")
    res5: Option[Array[Byte]] = None

## Using Client Pooling

scala-redis is a blocking client, which serves the purpose in most of the cases since Redis is also single threaded. But there may be situations when clients need to manage multiple RedisClients to ensure thread-safe programming.

scala-redis includes a Pool implementation which can be used to serve this purpose. Based on Apache Commons Pool implementation, RedisClientPool maintains a pool of instances of RedisClient, which can grow based on demand. Here's a sample usage ..

    val clients = new RedisClientPool("localhost", 6379)
    def lp(msgs: List[String]) = {
      clients.withClient {
        client => {
          msgs.foreach(client.lpush("list-l", _))
          client.llen("list-l")
        }
      }
    }

Using a combination of pooling and futures, scala-redis can be throttled for more parallelism. This is the typical recommended strategy if you are looking forward to scale up using this redis client. Here's a sample usage .. we are doing a parallel throttle of an lpush, rpush and set operations in redis, each repeated a number of times ..

If we have a pool initialized, then we can use the pool to repeat the following operations. 

    // lpush
    def lp(msgs: List[String]) = {
      clients.withClient {
        client => {
          msgs.foreach(client.lpush("list-l", _))
          client.llen("list-l")
        }
      }
    }

    // rpush
    def rp(msgs: List[String]) = {
      clients.withClient {
        client => {
          msgs.foreach(client.rpush("list-r", _))
          client.llen("list-r")
        }
      }
    }

    // set
    def set(msgs: List[String]) = {
      clients.withClient {
        client => {
          var i = 0
          msgs.foreach { v =>
            client.set("key-%d".format(i), v)
            i += 1
          }
          Some(1000)
        }
      }
    }

And here's the snippet that throttles our redis server with the above operations in a non blocking mode using Scala futures:

    val l = (0 until 5000).map(_.toString).toList
    val fns = List[List[String] => Option[Int]](lp, rp, set)
    val tasks = fns map (fn => scala.actors.Futures.future { fn(l) })
    val results = tasks map (future => future.apply())

## Redis with Sentinel

Redis client can be managed by a sentinel cluster. A sentinel cluster can be initialized either by the list of sentinel addresses in the cluster

    val sentinels = Set(
          SentinelAddress("sandbox-sentinel-i-dd71f533.use1a.apelabs.net", 26379),
          SentinelAddress("sandbox-sentinel-i-7c877c92.use1a.apelabs.net", 26379),
          SentinelAddress("sandbox-sentinel-i-7f877c91.use1a.apelabs.net", 26379))


    SentinelClusterBuilder(SentinelClusterConfig(), sentinels)

or a load balancer of sentinel cluster

    SentinelClusterBuilder(SentinelClusterConfig(), SentinelAddress("internal-sandbox-sentinel-test-1813530000.us-east-1.elb.amazonaws.com", 26379), Set("test-service", "test-service-2"))

If the sentinel cluster is initialized by a load balancer, you must provide the master names managed in this cluster so that sentinel cluster client can auto discover all sentinel instances behind the load balancer.

Once a sentinel cluster is created, you could create standalone redis client

     new RedisClientBySentinel("test-service", sentinelCluster, (redisNode: RedisNode) => {})

or a redis client pool

     new RedisPoolStorage(new RedisClientPoolBySentinel(configuration.name, sentinelCluster))

or a redis client cluster

     new RedisShards(clusteredHosts) with NoOpKeyTagPolicy with PoolCreationBySentinel

## License

This software is licensed under the Apache 2 license, quoted below.

Licensed under the Apache License, Version 2.0 (the "License"); you may not
use this file except in compliance with the License. You may obtain a copy of
the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
License for the specific language governing permissions and limitations under
the License.

