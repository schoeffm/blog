+++
title = "jq by example"
date = 2020-01-29T15:07:59+01:00
tags = ["cli", "jq", "json", "tools", "scripting", "bash" ]
+++

Inspired by a [nice tutorial on Baeldung][baeldung] I've decided to write done some usual suspects when working with [jq][jq]. Hence, this is more like a cheatsheet than a regular post.

For practitioners and interactive learners I also recommend the great [jq-play online editor][editor] where you can try [jq][jq] directly in the browser.

For our examples below let's go with the followin' JSON-input (fetched from a micro-profile `/health`-endpoint).

{{< highlight json>}}
{
    "outcome": "UP",
        "checks": [
        {
            "name": "core-isAlive",
            "state": "UP",
            "data": {
                "buildtime": "2020-01-29T12:46:05.183Z",
                "version": "0.0.1-SNAPSHOT",
                "api-versions": "2"
            }
        },
        {
            "name": "core-db2-isAlive",
            "state": "UP",
            "data": {}
        },
        {
            "name": "core-postgres-isAlive",
            "state": "UP",
            "data": {}
        }]
}
{{< /highlight >}}

In the subsequent examples I use `jq` for brevity reasons with a static file-input: `jq '.' file.json`
But normally I'd use it rather by piping a response directly into it: `curl -s http://localhost:8080/health | jq '.'`

### Queries and their results
Extracting a property(-value) - here the `checks`-property:
{{< highlight bash>}}
> jq '.checks' file.json
{{< /highlight >}}
Process individual array-items (and extract props there):
{{< highlight bash>}}
> jq '.checks[] | .name' file.json
"core-isAlive"
"core-db2-isAlive"
"core-postgres-isAlive"
{{< /highlight >}}
Notice the difference when you specifiy `raw`-output (`-r`):
{{< highlight bash>}}
> jq -r '.checks[1] | .name' file.json
core-db2-isAlive
{{< /highlight >}}
Determining the length of arrays or prop-values:
{{< highlight bash>}}
> jq '.checks | lengh' file.json
3
> jq '.checks[0].name | length' file.json
12
{{< /highlight >}}
Extract only keys of an object/map (using `-c` for compact output): 
{{< highlight bash>}}
> jq '.checks[0] | keys' file.json -c 
["data","name","state"]
> jq '.checks | keys' file.json -c
[0,1,2]
{{< /highlight >}}
Finding/Selecting entries by value (using `==`, `>=`, `<` etc.) or regex (using the `test`-built-in filter):
{{< highlight bash>}}
> jq '.checks[] | select(.state == "UP") | select (.name|test("post"))' file.json
{
    "name": "core-postgres-isAlive",
    "state": "UP",
    "data": {}
}
{{< /highlight >}}
If a prop contains special characters you have to quote the name:
{{< highlight bash>}}
> jq '.checks[] | select(.data."api-versions"=="2")' file.json
{
    "name": "core-isAlive",
        "state": "UP",
        "data": {
            "buildtime": "2020-01-29T12:46:05.183Z",
            "version": "0.0.1-SNAPSHOT",
            "api-versions": "2"
        }
}

{{< /highlight >}}
Slicing works by indexing the start/end-point `2:3` (first inclusive, second exclusive) - you can also omit one of the indices i.e. `:3`:
{{< highlight bash>}}
> jq '.checks[2:3]' file.json
{{< /highlight >}}
Change the structure of the output by introducing new objects/arrays:
{{< highlight bash>}}
  # outermost brakets form a new array - inner curlies create new objects
> jq '[.checks[] | { service: .name, status: .state }]' file.json
[
{
    "service": "core-isAlive",
    "status": "UP"
},
{
    "service": "core-db2-isAlive",
    "status": "UP"
},
{
    "service": "core-postgres-isAlive",
    "status": "UP"
}
]
{{< /highlight >}}
You can also define the value of a prop as the key in a new structure:
{{< highlight bash>}}
> jq '.checks[1] | { (.name): .state }' file.json -c
{"core-db2-isAlive":"UP"}
{{< /highlight >}}
Using `+` you can combine arrays, strings, numbers and even objects:
{{< highlight bash>}}
> jq '.checks[1] | { (.name): .state } + { "plus": (.state +"__"+ .name) }' file.json
{
    "core-db2-isAlive": "UP",
    "plus": "UP__core-db2-isAlive"
}
{{< /highlight >}}
`join` is useful for preparing scripting-input (also `-c` is handy here):
{{< highlight bash>}}
> jq '[.checks[] | .name ] | join(" ")' -c file.json
core-isAlive core-db2-isAlive core-postgres-isAlive
{{< /highlight >}}
Use `add` in case you'd like to sum things up - to convert types use `tonumber` or `tostring` etc.:
{{< highlight bash>}}
> jq '[.checks[] | .data?."api-versions" | select(. != null) | tonumber ] | add' -c file.json
2
{{< /highlight >}}
`split` in combination with `first` and `last` are handy when extracting things from property-values itself:
{{< highlight bash>}}
> # we'd like to extract only the second part of buildtime '2020-01-29T12:46:05.183Z'
> jq '.checks[].data.buildtime | select(. != null) | split("T") | last' file.json
12:46:05.183Z
{{< /highlight >}}

[baeldung]:https://www.baeldung.com/linux/jq-command-json
[jq]:https://stedolan.github.io/jq/manual/
[editor]:https://jqplay.org/
