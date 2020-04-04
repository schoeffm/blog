+++
title = "Towards Flexible Metrics Using SimplyTimed"
date = 2020-03-29T18:35:36+02:00
tags = ["metrics", "prometheus", "promql", "microprofile", "quarkus", "grafana"]
draft = true
+++

When [revision 2.3 of the microprofile metrics API][metrics-release] was release earlier this year it introduced a new metric type called __*Simple Timer*__. Granted, with a name like that it's easy to underestimate this new type - but don't get tricked by its unspectacular title. Turns out, most of the time (especially in my projects where we also make use of [Prometheus][prometheus]) it's a way better suited and more flexible metric than i.e. its older cousin __*Timer*__.
<!--more-->
### `@SimplyTimed` - why should I care

The [specification itself][spec-arch] describes the __*Simple Timer*__ as:

> a lightweight alternative to the timer metric that only tracks the elapsed time duration and invocation counts. The simple timer may be preferrable over the timer when used with Prometheus as the statistical calculations can be deferred to Prometheus using the simple timer’s available values.

which already explains why it is a better fit in a [Prometheus][prometheus]-based setup. It only comprises __two__ metrics instead of the __fifteen__ provided by each __*Timer*__. Nevertheless, with these two metrics and the usage of [PromQL][promql] you're still able to extract a bunch of information.

The issue with __*Timer*__ is that it offers already aggregated values like rates, standard-deviation or mean-values without giving you the raw, underlying data used for those aggregations. Moreover, those aggregates are provided independently by each microservice instance and it's problematic to aggregate those instance-specific values up to get a service-wide view - or as Brian Brazil stated it in his phantastic book [Prometheus - Up and Running][upandrunning]

> you could end up averaging averages, which is not statistically valid.

### Sounds nice - can I see it action?

[I've created a small setup][repo] where [Prometheus][prometheus] scrapes three instances of the same microservice (configured to show different performance behaviour). For easier consumption the scraped data can be visualized by an also included [Grafana][grafana] dashboard.

{{< figure src="/posts/metrics-with-microprofile/setup.png" alt="example setup for scraping three microservice instances">}}

You can build and start the whole stack using the scripts included in [the repository][repo]:

{{< highlight bash>}}
$> ./buildAndRun.sh && \      # compile, build and start everyth.
   ./produceSteadyLoad.sh     # put some traffic on the services

$> # Grafana:    http://localhost:3000
$> # Prometheus: http://localhost:9090

{{< /highlight >}}

To setup Grafana you'll have [to login first](http://localhost:3000) (using _admin/admin_ and skip any further user-setup). The repository contains a pre-build dashboard you can import - to do this, follow these preparation steps:

{{< figure src="/posts/metrics-with-microprofile/grafana_setup.png" alt="Steps to configure grafana properly">}}

* first, add a new prometheus data source (step 1 and 2)
* the dockerized services can talk to each other via their service names - so prometheus is accessable at http://prometheus:9090 - confirm this dialog via hitting `Enter` or the save button at the bottom (step 3)
* finally, import the `grafana_dashboard.json` contained in the referenced repository (triggered via 4) 

### Finally, we can play with the data

Well, let's see what the `@SimplyTimed`-annotation produces - as I've mentioned above there are only two metrics included. For my `HelloResource`-class and its `simplyTimedHello`-method this results in:

```bash
application_de_bender_metrics_HelloResource_simplyTimedHello_elapsedTime_seconds
application_de_bender_metrics_HelloResource_simplyTimedHello_total
```

The first one being a gauge (representing the total amount of seconds) while the second one is a simple counter (representing the total number of calls). Both values will constantly increase and are of minor use in their raw form. 

{{< figure src="/posts/metrics-with-microprofile/01_prometheus.png" alt="">}}

To make 'em more expressive we can apply the `rate`-function of prometheus. This will show us the change-rate of that specific metric.

{{< figure src="/posts/metrics-with-microprofile/02_prometheus.png" alt="">}}

Thought certainly nice, the value of this graph is still doubtful - but represents the foundation of our next step (which is much more useful).

##### Average response performance (per instance)

As with ordinary average-calculations we can divide _the rated_ `elapsedTime` by the _the rated_ `total` amount of calls:

```bash
rate(application_de_bender_metrics_HelloResource_simplyTimedHello_elapsedTime_seconds[3m])
/ 
rate(application_de_bender_metrics_HelloResource_simplyTimedHello_total[3m])
```

This division matches the time series with the same labels and divides 'em - resulting in the same three time series stating the average response time over the past 3 minutes as a value:

```bash
{instance="service_b:8080",job="service_b"} 0.17564152650602402
{instance="service_a:8080",job="service_a"} 0.09243990722891576
{instance="service_c:8080",job="service_c"} 0.7002928650602409
```

{{< figure src="/posts/metrics-with-microprofile/03_prometheus.png" alt="">}}

Now, this representation has real value since it shows us how each microservice is doing in terms of response performance. In our artificial setup one microservice instance performs much worse than the other two instances.

That value by the way serves pretty much the same purpose as __*Timers*__ `*_mean_seconds` metrics (__Notice:__ mean and average [aren't necessarily the same thing][meanvsaverage]).

Still, in a microservice landscape a holistic view to the service (not the individual instance) would be more appreciated. 

##### Average response performance (per service)

So, the query from above will produce a result per label set (i.e. `service_a` != `service_b`) and thus per individual instance.
Well, if we would now remove that ambiguity by dumping all labels that differentiate the metrics and sum those values up we'd end up in one, aggregated result:

```bash
sum without(instance,job) 
  (rate( application_de_bender_metrics_HelloResource_simplyTimedHello_elapsedTime_seconds[3m]))
/ 
sum without(instance,job) 
  (rate(application_de_bender_metrics_HelloResource_simplyTimedHello_total[3m]))  
```

And voilà:

```bash
{}  0.32279171354972847
```

{{< figure src="/posts/metrics-with-microprofile/04_prometheus.png" alt="">}}

This now represents the average performance of the overall service. That's also the avg. performance a random user can expect who cannot simply select the most performant service instance but whose request will randomly be routed to one of the cluster members. A nice representative of how well our service is doin'.

Oh and by the way, that value is not reproducable using __*Timer*__ (without stumbling upon the _average of averages issue_) since it lacks an `_elapsedTime_`-pendant.

### Conclusion

__*Simple Timer*__ isn't just a smaller, simpler version of __*Timer*__. In combination with Prometheus it can even be of more use than it's older brother since it allows us to aggregate values accross JVM-borders (IMHO not applicable to __*Timer*__) which is probably its greatest benefit.

[metrics-release]:https://github.com/eclipse/microprofile-metrics/releases
[spec-arch]:https://download.eclipse.org/microprofile/microprofile-metrics-2.3/microprofile-metrics-spec-2.3.html#meta-data-def
[prometheus]:https://prometheus.io/
[promql]:https://prometheus.io/docs/prometheus/latest/querying/basics/
[grafana]:https://grafana.com/oss/grafana/
[upandrunning]:https://www.oreilly.com/library/view/prometheus-up/9781492034131/
[meanvsaverage]:http://www.differencebetween.net/science/difference-between-average-and-mean/
[repo]:https://github.com/schoeffm/metrics-with-microprofile
