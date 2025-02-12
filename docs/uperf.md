# Uperf

[Uperf](http://uperf.org/) is a network performance tool

## Running UPerf

Given that you followed instructions to deploy operator,
you can modify [cr.yaml](../resources/crds/ripsaw_v1alpha1_uperf_cr.yaml)

```yaml
apiVersion: ripsaw.cloudbulldozer.io/v1alpha1
kind: Benchmark
metadata:
  name: uperf-benchmark
  namespace: ripsaw
spec:
  workload:
    name: uperf
    args:
      hostnetwork: false
      pin: false
      pin_server: "node-0"
      pin_client: "node-1"
      samples: 1
      pair: 1
      test_types:
        - stream
      protos:
        - tcp
      sizes:
        - 16384
      runtime: 30
```

`hostnetwork` will test the performance of the node the pod will run on.

*Note:* If you want to run with hostnetwork on `OpenShift`, you will need to execute the following:

```bash

$ oc adm policy add-scc-to-user privileged -z benchmark-operator

```

`pin` will allow the benchmark runner place nodes on specific nodes, using the `hostname` label.

`pin_server` what node to pin the server pod to.

`pin_client` what node to pin the client pod to.

`samples` how many times to run the tests. For example

```yaml
      samples: 3
      pair: 1
      test_types:
        - stream
      protos:
        - tcp
      sizes:
        - 1024
        - 16384
      runtime: 30
```

Will run `stream` w/ `tcp` and message size `1024` three times and
`stream` w/ `tcp` and message size `16384` three times. This will help us
gain confidence in our results.

Once done creating/editing the resource file, you can run it by:

```bash
# kubectl apply -f resources/crds/ripsaw_v1alpha1_uperf_cr.yaml # if edited the original one
# kubectl apply -f <path_to_file> # if created a new cr file
```

# Storing results into Elasticsearch

```yaml
apiVersion: ripsaw.cloudbulldozer.io/v1alpha1
kind: Benchmark
metadata:
  name: example-benchmark
  namespace: ripsaw
spec:
  test_user: test_user # user is a key that points to user triggering ripsaw, useful to search results in ES
  elasticsearch:
    server: <es_host>
    port: <es_port>
  workload:
    name: uperf
    args:
      hostnetwork: false
      pin: false
      pin_server: "node-0"
      pin_client: "node-1"
      samples: 1
      pair: 1
      test_types:
        - stream
      protos:
        - tcp
      sizes:
        - 16384
      runtime: 30
```

The new fields :

`elasticsearch.server` this is the elasticsearch cluster ip you want to send the result data to for long term storage.

`elasticsearch.port` port which elasticsearch is listening, typically `9200`.

`user` provide a user id to the metadata that will be sent to Elasticsearch, this makes finding the results easier.

By default we will utilize the `uperf-results` index for Elasticsearch.

Deploying the above(assuming pairs is set to 1) would result in

```bash
# kubectl get -o wide pods
NAME                                                    READY   STATUS      RESTARTS   AGE     IP             NODE       NOMINATED NODE   READINESS GATES
benchmark-operator-6679867fb7-p2fzb                     2/2     Running     0          6h1m    10.130.0.56    master-2   <none>           <none>
uperf-benchmark-nohost-uperf-client-10.128.1.29-kbw4b   0/1     Completed   0          3h11m   10.129.1.214   master-1   <none>           <none>

```

The first pod is our Operator orchestrating the UPerf workload.

To review the results, `kubectl logs <client>`, the top of the output is
the actual workload that was passed to UPerf (From the values in the custom resource).

Note: If cleanup is not set in the spec file then the client pods will be killed after
600 seconds from it's completion. The server pods will be cleaned up immediately
after client job completes

```
... Trimmed output ...
+-------------------------------------------------- UPerf Results --------------------------------------------------+
Run : 1
Uperf Setup

          hostnetwork : False
          client: 10.129.1.214
          server: 10.128.1.29

UPerf results for :

          test_type: stream
          protocol: tcp
          message_size: 64

UPerf results (bytes/sec):

          min: 0
          max: 75938816
          median: 72580096.0
          average: 63342843.6066
          95th: 75016192.0
+-------------------------------------------------------------------------------------------------------------------+

```

# Dashboard example

Using the Elasticsearch storage describe above, we can build dashboards like the below.

![UPerf Dashboard](https://i.imgur.com/gSVZ9MX.png)

To reuse the dashboard above, use the json [here](https://github.com/cloud-bulldozer/arsenal/tree/master/uperf/grafana)

Additionally, by default we will utilize the `uperf-results` index for Elasticsearch.
