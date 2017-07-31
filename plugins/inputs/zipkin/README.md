# Zipkin Plugin

This plugin implements the Zipkin http server to gather trace and timing data needed to troubleshoot latency problems in microservice architectures.



### Configuration:
```toml
[[inputs.zipkin]]
    path = "/api/v1/spans" #Path on which Telegraf listens for spans
    port = 9411 # Port on which Telegraf listens
```

### Tracing:

This plugin uses Annotations tags and fields to track data from spans

- TRACE : is a set of spans that share a single root span.
Traces are built by collecting all Spans that share a traceId.

- SPAN : is a set of Annotations and BinaryAnnotations that correspond to a particular RPC.

- Annotations : for each annotation & binary annotation of a span a metric is output


#### Annotations: records an occurrence in time at the beginning and end of a request
    - CS (client start) : beginning of span, request is made.
    - SR (server receive): server receives request and will start processing it
      network latency & clock jitters differ it from cs
    - SS (server send) : server is done processing and sends request back to client
      amount of time it took to process request will differ it from sr
    - CR (client receive): end of span, client receives response from server
      RPC is considered complete with this annotation

- TAGS:
  * __"id":__               The 64 bit ID of the span.
  * __"parent_id":__        An ID associated with a particular child span.  If there is no child span, the parent ID is set to ID.
  * __"trace_id":__        The 64 or 128-bit ID of a particular trace. Every span in a trace shares this ID. Concatenation of high and low and converted to hexadecimal.
  * __"name":__             Defines a span
  * __"service_name":__     Defines a service
  * __"annotation":__       The value of an annotation
  * __"endpoint_host":__    Listening port concat with IPV4

-FIELDS
      "duration_ns":             The time in nanoseconds between the end and beginning of a span.

### BINARY ANNOTATIONS:

-TAGS: Contains the same tags as annotations plus these additions

      "annotation_key": label describing the annotation

-FIELDS:

      "duration_ns": The time in nanoseconds between the end and beginning of a span.



### Sample Queries:

__Get All Span Names for Service__ `my_web_server`
```sql
SHOW TAG VALUES FROM "zipkin" with key="name" WHERE "service_name" = 'my_web_server' 
```
  - _Description:_  returns a list containing the names of the spans which have annotations with the given `service_name` of `my_web_server`.

__Get All Service Names__
```sql
SHOW TAG VALUES FROM "zipkin" WITH KEY = "service_name"
```
  - _Description:_  returns a list of all `distinct` endpoint service names.

__Find spans with longest duration__
```sql
SELECT max("duration_ns") FROM "zipkin" WHERE "service_name" = 'my_service' AND "name" = 'my_span_name' AND time > now() - 20m GROUP BY "trace_id",time(30s) LIMIT 5
```
  - _Description:_  In the last 20 minutes find the top 5 longest span durations for service `my_server` and span name `my_span_name`



### Example Input Trace:

- [Cli microservice with two services Test](https://github.com/openzipkin/zipkin-go-opentracing/tree/master/examples/cli_with_2_services)
- [Test data from distributed trace repo sample json](https://github.com/mattkanwisher/distributedtrace/blob/master/testclient/sample.json)

#### Trace Example
```{
      "traceId": "bd7a977555f6b982",
      "name": "query",
      "id": "be2d01e33cc78d97",
      "parentId": "ebf33e1a81dc6f71",
      "timestamp": 1458702548786000,
      "duration": 13000,
      "annotations": [
        {
          "endpoint": {
            "serviceName": "zipkin-query",
            "ipv4": "192.168.1.2",
            "port": 9411
          },
          "timestamp": 1458702548786000,
          "value": "cs"
        },
        {
          "endpoint": {
            "serviceName": "zipkin-query",
            "ipv4": "192.168.1.2",
            "port": 9411
          },
          "timestamp": 1458702548799000,
          "value": "cr"
        }
      ],
      "binaryAnnotations": [
        {
          "key": "jdbc.query",
          "value": "select distinct `zipkin_spans`.`trace_id` from `zipkin_spans` join `zipkin_annotations` on (`zipkin_spans`.`trace_id` = `zipkin_annotations`.`trace_id` and `zipkin_spans`.`id` = `zipkin_annotations`.`span_id`) where (`zipkin_annotations`.`endpoint_service_name` = ? and `zipkin_spans`.`start_ts` between ? and ?) order by `zipkin_spans`.`start_ts` desc limit ?",
          "endpoint": {
            "serviceName": "zipkin-query",
            "ipv4": "192.168.1.2",
            "port": 9411
          }
        },
        {
          "key": "sa",
          "value": true,
          "endpoint": {
            "serviceName": "spanstore-jdbc",
            "ipv4": "127.0.0.1",
            "port": 3306
          }
        }
      ]
    },
    ```

### Recommended installation

We recomend using the [tsi influxDB engine](https://www.influxdata.com/path-1-billion-time-series-influxdb-high-cardinality-indexing-ready-testing/) as it can accept high cardinality data.
#### How To Set Up InfluxDB For Work With Zipkin

##### Steps
___Update___ InfluxDB to >= 1.3, in order to use the new tsi engine.

___Generate___ a config file with the following command:
    `influxd config > /path/for/config/file`

___Add___ the following to your config file, under the `[data]` tab:

```toml
[data]
    dir = "/Users/goller/.influxdb/data"
    index-version = "tsi1"
    wal-dir = "/Users/goller/.influxdb/wal"
    query-log-enabled = true
    cache-max-memory-size = 1073741824
    cache-snapshot-memory-size = 26214400
    cache-snapshot-write-cold-duration = "10m0s"
    compact-full-write-cold-duration = "4h0m0s"
    max-series-per-database = 1000000
    max-values-per-tag = 100000
    trace-logging-enabled = false
 ```

 ___Start___ `influxd` with your new config file:
 `$ influxd -config=/path/to/your/config/file`

___Update___ your retention policy:
```sql
ALTER RETENTION POLICY "autogen" ON "telegraf" DURATION 1d SHARD DURATION 30m
```