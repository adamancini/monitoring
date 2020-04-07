# monitoring
robust docker swarm &amp; kubernetes monitoring &amp; logging infrastructure

---


#### install netdata on all cluster nodes

`dci cluster ssh ee21 "bash <(curl -Ss https://my-netdata.io/kickstart-static64.sh) --dont-wait"`

> if RHEL/CentOS configure cgroup accounting per:
- https://docs.netdata.cloud/collectors/cgroups.plugin/#monitoring-systemd-services

#### cadvisor

`dci cluster ssh ee21 "docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  google/cadvisor:latest"`

#### set up prometheus
> https://docs.docker.com/ee/ucp/admin/configure/collect-cluster-metrics/

UCP ships with a Prometheus DaemonSet configured in k8s - we can patch this daemonset to run an HA configuration on worker nodes

Label at least two of your worker nodes with the kv pair: `ucp-metrics=`  the value is blank.  Patch the ucp-metrics DaemonSet's nodeSelector using teh same key and value used for the node label:


```
$ kubectl label no ip-172-31-13-146.us-east-2.compute.internal ip-172-31-15-60.us-east-2.compute.internal ucp-metrics=
node/ip-172-31-13-146.us-east-2.compute.internal labeled
node/ip-172-31-15-60.us-east-2.compute.internal labeled

$ kubectl -n kube-system patch daemonset ucp-metrics --type json -p '[{"op": "replace", "path": "/spec/template/spec/nodeSelector", "value": {"ucp-metrics": ""}}]'

$ kubectl -n kube-system get po -l k8s-app=ucp-metrics -o wide
NAME                READY   STATUS        RESTARTS   AGE   IP               NODE                                          NOMINATED NODE
ucp-metrics-9wbwr   3/3     Running       0          31s   192.168.51.193   ip-172-31-15-60.us-east-2.compute.internal    <none>
ucp-metrics-h96st   3/3     Terminating   3          8d    192.168.57.198   ip-172-31-8-200.us-east-2.compute.internal    <none>
ucp-metrics-t7b7f   3/3     Running       0          31s   192.168.156.1    ip-172-31-13-146.us-east-2.compute.internal   <none>
```

Obtain an admin bundle and create a k8s secret containing your bundle's TLS certificates:

` (cd $DOCKER_CERT_PATH && kubectl create secret generic prometheus --from-file=ca.pem --from-file=cert.pem --from-file=key.pem)`

then create a prometheus deployment and ClusterIP service:

```
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
 name: prometheus
 namespace: kube-system
data:
 prometheus.yaml: |
   global:
     scrape_interval: 10s
   scrape_configs:
   - job_name: 'ucp'
     tls_config:
       ca_file: /bundle/ca.pem
       cert_file: /bundle/cert.pem
       key_file: /bundle/key.pem
       server_name: proxy.local
     scheme: https
     file_sd_configs:
     - files:
       - /inventory/inventory.json
---
apiVersion: apps/v1
kind: Deployment
metadata:
 name: prometheus
spec:
 replicas: 2
 selector:
   matchLabels:
     app: prometheus
 template:
   metadata:
     labels:
       app: prometheus
   spec:
     containers:
     - name: inventory
       image: alpine
       command: ["sh", "-c"]
       args:
       - apk add --no-cache curl &&
         while :; do
           curl -Ss --cacert /bundle/ca.pem --cert /bundle/cert.pem --key /bundle/key.pem --output /inventory/inventory.json https://ucp-controller.kube-system.svc.cluster.local/metricsdiscovery;
           sleep 15;
         done
       volumeMounts:
       - name: bundle
         mountPath: /bundle
       - name: inventory
         mountPath: /inventory
     - name: prometheus
       image: prom/prometheus
       command: ["/bin/prometheus"]
       args:
       - --config.file=/config/prometheus.yaml
       - --storage.tsdb.path=/prometheus
       - --web.console.libraries=/etc/prometheus/console_libraries
       - --web.console.templates=/etc/prometheus/consoles
       volumeMounts:
       - name: bundle
         mountPath: /bundle
       - name: config
         mountPath: /config
       - name: inventory
         mountPath: /inventory
     volumes:
     - name: bundle
       secret:
         secretName: prometheus
     - name: config
       configMap:
         name: prometheus
     - name: inventory
       emptyDir:
         medium: Memory
---
apiVersion: v1
kind: Service
metadata:
 name: prometheus
spec:
 ports:
 - port: 9090
   targetPort: 9090
 selector:
   app: prometheus
 sessionAffinity: ClientIP
EOF
 ```

Check that the application is reachable on the ClusterIP using an ssh reverse tunnel:

`ssh -L 9090:10.96.189.99:9090 ubuntu@ec2-3-17-187-118.us-east-2.compute.amazonaws.com`

open browser, navigate to 127.0.0.1:9090, see prometheus dashboard



##### supporting material

GOTO 2017 • How Prometheus Revolutionized Monitoring at SoundCloud • Björn Rabenstein
> https://www.youtube.com/watch?v=hhZrOHKIxLw&feature=youtu.be

Monitoring, the Prometheus Way
> https://www.youtube.com/watch?v=PDxcEzu62jk

How to Export Prometheus Metrics from Just About Anything - Matt Layher, DigitalOcean
>https://www.youtube.com/watch?v=Zk09Mbu0YQk

clemenko/prometheus
> https://github.com/clemenko/prometheus

kernel sysctl tunables
> https://www.emc.com/collateral/technical-documentation/fine-tuning-scaleio-performance.pdf
> https://blog.codeship.com/running-1000-containers-in-docker-swarm/

swarmstack/swarmstack
> https://github.com/swarmstack/swarmstack

finding consul services w/ prom
> https://www.robustperception.io/finding-consul-services-to-monitor-with-prometheus

updated netdata & prom & grafana stack
> https://docs.netdata.cloud/backends/walkthrough/
