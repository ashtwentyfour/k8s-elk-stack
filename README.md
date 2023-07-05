# ELK Stack Deployment on K8s

## Deploying Elasticsearch Pods

* Create a new Namespace:

```
% kubectl create ns elk
namespace/elk created
```

* Deploy Elasticsearch on the first node and to begin bootstrapping the cluster:

```
% kubectl apply -f elasticsearch/bootstrap-service.yml
service/es-node-01 created
% kubectl apply -f elasticsearch/bootstrap-pvc.yml    
persistentvolumeclaim/pvc-bootstrap created
% kubectl apply -f elasticsearch/bootstrap-pod.yml 
pod/es-node-01 created
```

* Wait for Pod ```es-node-01``` to start Elasticsearch before deploying the 2nd master node and the first data node:

```
% kubectl apply -f elasticsearch/master-02-service.yml
service/es-node-02 created
% kubectl apply -f elasticsearch/master-02-pvc.yml    
persistentvolumeclaim/pvc-master-02 created
% kubectl apply -f elasticsearch/master-02-deploy.yml 
deployment.apps/es-node-02 created

% kubectl apply -f elasticsearch/data-01-service.yml  
service/es-data-01 created
% kubectl apply -f elasticsearch/data-01-pvc.yml    
persistentvolumeclaim/pvc-data-01 created
% kubectl apply -f elasticsearch/data-01-deploy.yml 
deployment.apps/es-data-01 created

% kubectl get pods -n elk                          
NAME                          READY   STATUS    RESTARTS   AGE
es-data-01-ccffcb79b-7756x    1/1     Running   0         5m41s
es-node-01                    1/1     Running   0          16m
es-node-02-666d5dcc99-s7p58   1/1     Running   0          12m
```

* Master nodes - ```master-03``` and ```master-04``` and data node ```data-02``` can also be deployed in a similar fashion of the cluster has enough resources (The ```es-node-01``` Pod can be deleted if the additional master nodes are running)

* Deploy the Service ```elasticsearch-service```:

```
% kubectl apply -f elasticsearch/elasticsearch-service.yml 
service/elasticsearch created
```

## Deploy Logstash

* Deploy Logstash configured to with an Elasticsearch output and HTTP input (for Fluent Bit):

```
% kubectl apply -f elasticsearch/elasticsearch-service.yml 
service/elasticsearch created
% kubectl apply -f logstash/logstash-fluent-service.yml 
service/logstash created
% kubectl apply -f logstash/logstash-fluent-pvc.yml    
persistentvolumeclaim/pvc-logstash-01 created
% kubectl apply -f logstash/logstash-fluent-deploy.yml 
deployment.apps/logstash-fluent created
```

* Check the logs from the Logstash Pod and validate that the Logstash service has connected to Elasticsearch:

```
% kubectl logs logstash-fluent-86b96b78d4-snxlh -n elk -f

[INFO ] 2023-07-05 20:49:17.964 [Api Webserver] agent - Successfully started Logstash API endpoint {:port=>9600, :ssl_enabled=>false}
[INFO ] 2023-07-05 20:49:18.649 [Converge PipelineAction::Create<main>] Reflections - Reflections took 261 ms to scan 1 urls, producing 132 keys and 464 values
[INFO ] 2023-07-05 20:49:19.257 [Converge PipelineAction::Create<main>] javapipeline - Pipeline `main` is configured with `pipeline.ecs_compatibility: v8` setting. All plugins in this pipeline will default to `ecs_compatibility => v8` unless explicitly configured otherwise.
[INFO ] 2023-07-05 20:49:19.286 [[main]-pipeline-manager] elasticsearch - New Elasticsearch output {:class=>"LogStash::Outputs::ElasticSearch", :hosts=>["//elasticsearch:9200"]}
[INFO ] 2023-07-05 20:49:19.383 [[main]-pipeline-manager] elasticsearch - Elasticsearch pool URLs updated {:changes=>{:removed=>[], :added=>[http://elasticsearch:9200/]}}
[WARN ] 2023-07-05 20:49:19.518 [[main]-pipeline-manager] elasticsearch - Restored connection to ES instance {:url=>"http://elasticsearch:9200/"}
```

## Deploy Fluent Bit

* Fluent Bit can be deployed using the publicly available Helm [chart](https://github.com/isItObservable/Episode3--Kubernetes-Fluentbit#lets-install-fluentbit-to-go-trough-the-configuration)

```
% helm repo add fluent https://fluent.github.io/helm-charts
% helm install fluent-bit fluent/fluent-bit
```

* Update the Fluent Bit ConfigMap in order to apply the Logstash configuration (Logstash is set as the Output) and restart Fluent Bit:

```
% kubectl apply -f fluent-bit/fluent-bit-cm.yml 
configmap/fluent-bit configured
% kubectl rollout restart ds fluent-bit
daemonset.apps/fluent-bit restarted
```

* Validate that logs are being sent to Logstash:

```
% kubectl logs fluent-bit-64n59

[2023/07/05 21:11:13] [ info] [output:http:http.0] logstash.elk.svc.cluster.local:5044, HTTP status=200
ok
[2023/07/05 21:11:14] [ info] [output:http:http.0] logstash.elk.svc.cluster.local:5044, HTTP status=200
ok
[2023/07/05 21:11:15] [ info] [output:http:http.0] logstash.elk.svc.cluster.local:5044, HTTP status=200
ok
[2023/07/05 21:11:16] [ info] [output:http:http.0] logstash.elk.svc.cluster.local:5044, HTTP status=200
ok
```

## Deploy Kibana

* Deploy Kibana v7.17.10 (Uses the public Kibana Docker image):

```
% kubectl apply -f kibana/kibana-deploy.yml 
deployment.apps/kibana created
```

## ELK Validation

* Get a shell on one of the Elasticsearch Pods:

```
% kubectl exec -it es-node-01 -n elk -c es-node-01 -- /bin/bash 
```

* Check the cluster health and list all the nodes:

```
esuser@es-node-01:/etc/elasticsearch$ curl http://elasticsearch:9200
{
  "name" : "es-node-02",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "rwRcl1kXR8Sz5_ibY-Wilw",
  "version" : {
    "number" : "7.17.11",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "eeedb98c60326ea3d46caef960fb4c77958fb885",
    "build_date" : "2023-06-23T05:33:12.261262042Z",
    "build_snapshot" : false,
    "lucene_version" : "8.11.1",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}

esuser@es-node-01:/etc/elasticsearch$ curl 'http://elasticsearch:9200/_cat/health?v&pretty'
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1688592598 21:29:58  elasticsearch yellow          3         1      9   9    0    0        1             0                  -                 90.0%

esuser@es-node-01:/etc/elasticsearch$ curl 'http://elasticsearch:9200/_cat/nodes?v&pretty'
ip        heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
10.20.4.3           24          63  12    0.01    0.11     0.15 d         -      es-data-01
10.20.1.4           18          70   6    0.12    0.48     0.43 m         *      es-node-01
10.20.3.3           24          70   8    0.06    0.03     0.02 m         -      es-node-02
```

* List all the indices and validate that there is atleast one ```logstash-*``` index where the document count is increasing:

```
esuser@es-node-01:/etc/elasticsearch$ curl 'http://elasticsearch:9200/_cat/indices?v&pretty'
health status index                            uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .geoip_databases                 4n2VexpMTIGWJq7bMWqz8g   1   0         41            0     39.5mb         39.5mb
green  open   .apm-custom-link                 yoGpk95zTLatsxnYoXBdSw   1   0          0            0       227b           227b
green  open   .kibana_7.17.10_001              iWb8vl4JQC2eYuyU4sOx3w   1   0         15            0      2.3mb          2.3mb
green  open   .apm-agent-configuration         Yl69ktXdSVCtdOQWkYPMaA   1   0          0            0       227b           227b
yellow open   logstash-2023.07.05              s_SmnQlhTdOeBDaaxpWjEw   1   1      13787            0      6.5mb          6.5mb
green  open   .kibana_task_manager_7.17.10_001 jWVVjWtBR6-5kqLsT5TCYg   1   0         17          445    121.6kb        121.6kb
```
