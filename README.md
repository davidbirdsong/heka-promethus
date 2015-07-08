# heka-promethus

[mozilla heka](https://github.com/mozilla-services/heka) output plugin which exposes an endpoint for prometheus to scrape. The internals are heavily influenced by the [collectd_exporter](https://github.com/prometheus/collectd_exporter)

Metrics are created and registered with the Prometheus client using an immutable implementation of the ```Metric``` interface [NewConstMetric](http://godoc.org/github.com/prometheus/client_golang/prometheus#NewConstMetric)

The ```valuetype``` Heka field will serve to [```ValueType```](http://godoc.org/github.com/prometheus/client_golang/prometheus#ValueType) the metric to: 
- ```GaugeValue```
- ```CounterValue```
-  ```UntypedValue```

### Status Expiremental
As advertised the internal message format has changed from a native Heka message to a json payload. Filters need to be able to emit 100's of messages on timer_event. The author wasn't able to grok how to encode many metric events while still providing an expressive albeit arbitrary tagging of each metric on a single Heka message.

The json payload solves that and also means external stuff can be blindly forwarded too.

### Usage
Send a heka message to this output plugin w/ a json message Payload 
```json
[{
  "name": "foo_metric1",
  "help": "foo metric counts stuff",
  "value": 100, # will 
  "valuetype": "Counter",
  "labels": {
    "service": "hooman1",
    "role": "hooman_runner1"
  }
},
{
  "name": "foo_metric2",
  "help": "foo metric counts stuff",
  "value": 200,
  "expires": 20,
  "valuetype": "Counter",
  "labels": {
    "service": "hooman-2",
    "role": "hooman_runner-2"
  }
}
]
```

```lua
{
	Payload = <json>
	Fields = {
		metricType = 'scalar',
	}
}

```
use *scalar* for ```Fields.metricType``` for counters and gauges.

future types could specify Summaries and Histograms

Add the following ```toml``` to heka:
```toml
[prometheus_out]
type = "PrometheusOutput"
message_matcher = 'Logger == "Anything"' # anything to route the message properly here
Address = "127.0.0.1:9112"
encoder = "RstEncoder"
default_ttl = '15s' # applied to any metrics w/ no expires, defautls to 90s

```
(encoder is specified for error logging only )


curl the new prometheus in heka:
```
[david@foulplay ~]$ curl http://127.0.0.1:9112/metrics  
# [lots of prometheus boilerplate metrics suprressed]
#
#
# HELP hekagateway_msg_failed improperly formatted messages
# # TYPE hekagateway_msg_failed counter
# hekagateway_msg_failed 3
# # HELP hekagateway_msg_success properly formatted messages
# # TYPE hekagateway_msg_success counter
# hekagateway_msg_success 0
## HELP net_counters number of packets on network
## TYPE net_counters gauge
#net_counters{host="machine1",service="webapp"} 100
```
### Building
append the following to ```heka/cmake/plugin_loader.cmake```
```
hg_clone(https://bitbucket.org/ww/goautoneg default)
git_clone(https://github.com/prometheus/client_golang master)
git_clone(https://github.com/prometheus/procfs master)
git_clone(http://github.com/matttproud/golang_protobuf_extensions master)
git_clone(http://github.com/golang/protobuf master)
git_clone(https://github.com/prometheus/client_model master)
git_clone(http://github.com/beorn7/perks master)

add_external_plugin(git https://github.com/davidbirdsong/heka-promethus master)
```
### TODO
Add support for multi-value metric types
-  [Histogram](http://godoc.org/github.com/prometheus/client_golang/prometheus#NewConstHistogram)
-  [Summary](http://godoc.org/github.com/prometheus/client_golang/prometheus#NewConstSummary)
