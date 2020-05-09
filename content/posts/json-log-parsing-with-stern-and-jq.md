---
title: "JSON-Log Parsing With stern and jq"
date: 2020-05-07T20:41:26+02:00
tags: ["jq","stern","logging","cli","payara"]
---

In many projects we use JSON as log format since it inheritly bears structure and thus is easy to process by log-aggregators (i.e. [AWS CloudWatch][cloudwatch] autom. discovers fields from json just like [fluentd][fluentd]).
<!--more-->
And still, although those aggregators are live safers when skimming through tons of logs I'm often faster investigating an isolated issue using the good old log-tailing. While this was already a doubtful pleasure in the past it became a real pain now with JSON as output format.

In order to ease that pain and bring back the pleasure of bygone days I combine [jq][jq] (a CLI json-processor I already [wrote about here]({{< relref "jq-by-example.md" >}})) with [stern][stern] (a multi pod log tailing tool for kubernetes).

```bash
#          ┌─> Tells `stern` to output the logs as-is (without prepending the 
#          │   pod-name - which would only irritate `jq`)
#          │      ┌─> Common sub-string of pod-names to be tailed (be just as
#          │      │   precise as you have to be - i.e. 'foo' will tail all pods
#          │      │   that contain that string in their name) 
#          │      │          ┌─> start with the last 10 lines
#          │      │          │         ┌─> some logs are still not JSON ... find
#          │      │          │         │   a common marker that all JSON-logs have
#          │      │          │         │   in common
#        ┌─┴──┐ ┌─┴─────┐  ┌─┴─────┐ ┌─┴───────────┐
$> stern -o raw <POD-NAME> --tail 10 -i 'LogMessage' | \
     jq '. | (.ThreadID +" | "+ .LogMessage)'
#            └──┬──────────────────────────┘
#               └─> we extract only the 'TreadID' and 'LogMessage' props from 
#                   the original log, concatenate 'em intermediary structure
```

`stern` will tail the logs of all matching pods to stdout. That stream of logs will then be piped into `jq` to select and reformat the required information.

Using the flexibility of `jq` you can further add select-filters like:
```bash
$> stern -o raw <POD-NAME> --tail 10 -i 'LogMessage' | \
     jq 'select(.LogMessage|test("Exiting")) | .LogMessage'
#        └─┬───────────────────────────────┘
#          └─> regex-test to filter only those entries that 
#              contain the word 'Exiting' somewhere in the 
#              'LogMessage's value

$> stern -o raw <POD-NAME> --tail 10 -i 'LogMessage' | \
     jq 'select(.ThreadID=="84") | .LogMessage'
#         └─┬───────────────────┘
#           └─> outputs only those entries that have a 'ThreadID'
#               of '84' (this is an exact match)

```
These examples would directly apply to [Payara's json log format][payara] but can easily be adjusted to match different formats.


[fluentd]:https://www.fluentd.org/centralized_application_logging
[cloudwatch]:https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_AnalyzeLogData-discoverable-fields.html
[jq]:https://stedolan.github.io/jq/
[stern]:https://github.com/wercker/stern
[payara]:https://payara.gitbooks.io/payara-server/documentation/payara-server/logging/json-formatter.html

