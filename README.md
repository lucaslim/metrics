Metrics
=======

* A node.js port of codahale's metrics library: https://github.com/codahale/metrics

How to Use
----------

**Import Metrics**

```javascript
metrics = require('./../deps/metrics')
```

**Start a metrics Server**

```javascript
metricsServer = new metrics.Server(config.metricsPort || 9091);
```

**Create some metrics**

```javascript
var counterForThingA = new metrics.Counter
  , histForThingB = new metrics.Histogram
  , meterForThingC = new metrics.Meter
  , timerForThingD = new metrics.Timer;
```

**Add the metrics to the server**

```javascript
metricsServer.addMetric('thingA', counterForThingA);
metricsServer.addMetric('thingB', counterForThingB);
metricsServer.addMetric('thingC', counterForThingC);
metricsServer.addMetric('thingD', counterForThingD);
```


Advanced Usage
--------------
Typical production deployments have multiple node processes per server.  Rather than each process exposing metrics on different ports, it makes more sense to expose the metrics from the "master" process.  Writing a thin wrapper around this api to perform the process communication is trivial, with a message passing setup, the client processes could look something like this:

```javascript
var Metric = exports = module.exports = function Metrics(messagePasser, eventType) {
  this.messagePasser = messagePasser;
  this.eventType = eventType;
}

Metric.prototype.newMetric = function(type, eventType) {
  this.messagePasser.sendMessage({
    method: 'createMetric'
    , type: type
    , eventType: eventType
  }); 
}
Metric.prototype.forwardMessage = function(method, args) { 
  this.messagePasser.sendMessage({
    method: 'updateMetric'
    , metricMethod: method
    , metricArgs: args
    , eventType: this.eventType
  }); 
}

Metric.prototype.update = function(val) { return this.forwardMessage('update', [val]); }
Metric.prototype.mark = function(n) { return this.forwardMessage('mark', [n]); }
Metric.prototype.inc = function(n) { return this.forwardMessage('inc', [n]); }
Metric.prototype.dec = function(n) { return this.forwardMessage('dec', [n]); }
Metric.prototype.clear = function() { return this.forwardMessage('clear'); }
```

And the server side that implements createMetric and updateMetric could look something like this:

```javascript
{
  createMetric: function(msg) {
      if (metricsServer) {
        msg.type = msg.type[0].toUpperCase() + msg.type.substring(1)
        metricsServer.addMetric(msg.eventType, new metrics[msg.type]);
      }
   }
  updateMetric: function(msg) {
    if (metricsServer) {
    var namespaces = msg.eventType.split('.')
      , event = namespaces.pop()
      , namespace = namespaces.join('.');
    var metric = metricsServer.trackedMetrics[namespace][event];
    metric[msg.metricMethod].apply(metric, msg.metricArgs);
  }
}
```

How to Collect
--------------

Hit the server on your configured port and you'll get a json representation of your metrics.  You should collect these periodically to generate timeseries and monitor the health of your application.
