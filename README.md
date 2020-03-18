# vegeta-post-replay
Vegeta load testing scripts

[Vegeta](https://github.com/tsenart/vegeta) is a fantasic HTTP load testing tool and library written in Go. Here is a good article to get you started quickly: [Load Testing with Vegeta](https://www.scaleway.com/en/docs/vegeta-load-testing/).

## use cases
### a single requset without body
This is one of the most common use cases, i.e. you want to put load to a single endpoint for methods that do not carry request body, e.g. GET, HEAD, etc. You can simple put the url on the command line and set parameters.
```
echo 'GET http://localhost:8080' | vegeta attack -rate 5000 -duration 10m -header "Accept: application/json"
```

### multiple requests without body
This is the case when you have multipole request urls, e.g. there is a performance issue on the production server and you want to replay the request log. Vegeta supports a `http` format which resembles the plain-text HTTP messages. You can define target urls in a file and pass that target file to vegeta using `-target` option.

A sample target file `target.txt`:
```
GET http://goku:9090/path/to/dragon?item=ball
Accept: application/json

GET http://user:password@goku:9090/path/to
Accept: application/json

HEAD http://goku:9090/path/to/success
```

Then pass it to vegeta: `vegeta attack -rate 5000 -duration 10m -targets=target.txt`

### requests with body - target file way
In other use cases, we need to load test requests with bodies, e.g. POST, PUT, etc. You can do it using targets file as well. Each line with a request body file:

A sample target file `post_target.txt`:
```
POST http://goku:9090/path/to/dragon?item=ball
Content-Type: application/json
@/tmp/test1.json

POST http://goku:9090/path/to/dragon?item=ball
Content-Type: application/json
@/tmp/test2.json
```

Then pass it to vegeta: `vegeta attack -rate 5000 -duration 10m -targets=post_target.txt`.

### requests with body - json streaming way
But this is not convenient for the request log replay, e.g. you could have thousands of POST requests, each with a different body. Creating individual file for each POST body is quite ugly. Luckily, vegeta also supports a json format that allows getting streaming inputs programmatically. If you are only testing against one POST endpoint, this is a much more convenient way.

For example:
```
jq -ncM '{method: "GET", url: "http://goku", body: "Punch!" | @base64, header: {"Content-Type": ["text/plain"]}}' |
  vegeta attack -format=json -rate=100 | vegeta encode
```

Based on this, if you have a same POST endpoint, but different JSON bodies, you can try to stream each JSON body into the vegeta command.

Here is a quick example:
   * [request.log](request.log): this is the request log you want to replay, the last column is a JSON body (notice that the JSON string contains no space). You can also just give a list of JSON bodies, one per line and adjust `traffic_gen` script accordingly.
   * [traffic_gen](traffic_gen): reads each line from `request.log`, get the JSON bodies and pass to vegeta. Note that you need to have `jq` installed.
   * [traffic_play](traffic_play): the actual script that calls vegeta to send load. You can adjust the parameters.

Note:
   * In the sample scripts, we are sending traffic to Postman-Echo service with a low qps. Please don't send excessive traffic to them!
   * In the example, we assume the request body is in JSON, if you need to send in other text format, you can make adjustments as needed.
   * We are using awk to get the last column from the request log, so the JSON should not contain whitespaces. But you can preprocess the request log file and use a list of JSON bodies directly, just update the traffic_gen script as you want.

## Required Tools
### Vegeta
See "Install section in the [vegeta README](https://github.com/tsenart/vegeta/blob/master/README.md)"

### jq
See https://github.com/stedolan/jq/wiki/Installation

## Try the scripts
```
>./traffic_play
Requests      [total, rate, throughput]  6, 1.20, 0.80
Duration      [total, attack, wait]      4.999039725s, 4.998992444s, 47.281µs
Latencies     [mean, 50, 95, 99, max]    208.404113ms, 83.094232ms, 946.114123ms, 946.114123ms, 946.114123ms
Bytes In      [total, mean]              1356, 226.00
Bytes Out     [total, mean]              94, 15.67
Success       [ratio]                    66.67%
Status Codes  [code:count]               0:2  200:4 
```

## Final Words
Lastly, if the vegeta script cannot satisfy your needs, for example, you need to replay POST requests with different URLs and different bodies. You always have the option to use vegata go library and build your own client application.