# SOP – Prometheus long-term storage in S3 with Thanos

Keep recent metrics on the Prometheus PVC, store older data in S3, and see the full history in Grafana.

## How it works

Prometheus keeps only recent data on its PVC. A Thanos sidecar copies every 2-hour block to an S3 bucket, and Grafana reads both through Thanos Query, so dashboards show the full history.

![Architecture: Prometheus + Thanos + S3 + Grafana](images/architecture.png)

| Component | What it does | Where it runs |
| --- | --- | --- |
| Prometheus | Collects metrics; keeps recent data on its PVC | Existing pod (unchanged) |
| Thanos sidecar | Uploads each finished 2h block to S3; serves recent data to Thanos Query | Extra container inside the Prometheus pod |
| S3 bucket | Stores old blocks long-term | Ceph RGW / NetApp S3 (lab: SeaweedFS) |
| Thanos Store Gateway | Reads old blocks from S3 | New Deployment |
| Thanos Query | Asks sidecar + Store Gateway, merges and de-duplicates | New Deployment |
| Thanos Compactor | Compacts, downsamples and deletes old data in S3 | New StatefulSet (production) |
| Grafana | Shows dashboards using Thanos Query as its data source | Existing |

## Repository layout

```
.
├── README.md                     # this SOP
├── images/architecture.png       # diagram
└── manifests/
    ├── objstore.yml              # S3 settings (becomes the thanos-objstore Secret)
    ├── thanos-values.yaml        # Helm values that add the sidecar
    ├── thanos-store.yaml         # Store Gateway + Service
    ├── thanos-query.yaml         # Query + Service
    └── thanos-compactor.yaml     # Compactor (production)
```

## Before you start

Replace the lab values with yours.

| Item | Lab value | Production |
| --- | --- | --- |
| Namespace | `monitoring` | Where Prometheus runs |
| S3 endpoint | `192.168.1.10:8333` (http) | From storage team (usually https) |
| Bucket | `prometheus-longterm` | e.g. `prometheus-<cluster>` |
| Access / secret key | `labaccess` / `LabSecret12345` | Dedicated Thanos user |
| Local retention | `6h` | e.g. `30d` |
| Thanos image | `quay.io/thanos/thanos:v0.42.4` | Same, or from internal registry |

Checks:

- [ ] Prometheus installed with kube-prometheus-stack (`helm list -A`)
- [ ] Worker nodes have free disk and memory (each Thanos pod needs about 256–512 Mi)
- [ ] Pods can reach the S3 endpoint (test in Step 2)
- [ ] Change request approved (Prometheus restarts once in Step 3)

## Step 1 – S3 bucket

In production, ask the storage team for a bucket and keys, then skip to Step 2. In the lab, install SeaweedFS on a Rocky Linux VM.

**1.1 Install** (on the S3 VM)

```bash
curl -fLo seaweedfs.tar.gz https://github.com/seaweedfs/seaweedfs/releases/latest/download/linux_amd64.tar.gz
tar xzf seaweedfs.tar.gz
install -m 0755 weed /usr/local/bin/weed
restorecon -v /usr/local/bin/weed
weed version
```

**1.2 Create the S3 login** – `/etc/seaweedfs/s3.json`

```bash
mkdir -p /etc/seaweedfs /data/seaweedfs
```

```json
{
  "identities": [
    {
      "name": "thanos",
      "credentials": [{ "accessKey": "labaccess", "secretKey": "LabSecret12345" }],
      "actions": ["Admin", "Read", "Write", "List", "Tagging"]
    }
  ]
}
```

**1.3 Run as a service** – `/etc/systemd/system/seaweedfs.service`

```ini
[Unit]
Description=SeaweedFS S3 storage
After=network-online.target

[Service]
ExecStart=/usr/local/bin/weed server -dir=/data/seaweedfs -s3 -s3.port=8333 -s3.config=/etc/seaweedfs/s3.json
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now seaweedfs
firewall-cmd --permanent --add-port={8333,8888}/tcp && firewall-cmd --reload
```

**1.4 Create the bucket**

```bash
echo "s3.bucket.create -name prometheus-longterm" | weed shell
echo "s3.bucket.list" | weed shell
```

✅ `prometheus-longterm` is listed. Web view: `http://<S3-VM>:8888/buckets/`.

## Step 2 – S3 Secret in Kubernetes

**2.1 Test that pods can reach S3** (on master)

```bash
kubectl run s3test -n monitoring --rm -it --restart=Never --image=busybox:1.36 -- \
  wget -T 5 -S -O- http://192.168.1.10:8333/ 2>&1 | tail -3
```

✅ An HTTP reply such as `403 Forbidden` means it works. ❌ `download timed out` means fix the network first (see Troubleshooting).

**2.2 Edit** [`manifests/objstore.yml`](manifests/objstore.yml) with your endpoint, bucket and keys.

`insecure: true` is for plain http (lab only). `bucket_lookup_type: path` is needed when the endpoint is an IP address.

**2.3 Create the Secret**

```bash
kubectl create ns monitoring
kubectl create secret generic thanos-objstore -n monitoring --from-file=manifests/objstore.yml
kubectl get secret thanos-objstore -n monitoring
```

✅ The Secret is listed with `DATA 1`.

## Step 3 – Prometheus with the Thanos sidecar

The sidecar is added through the Helm values ([`manifests/thanos-values.yaml`](manifests/thanos-values.yaml)). The Prometheus Operator then adds a `thanos-sidecar` container to the Prometheus pod, which restarts once; the PVC and its data stay.

**3.1a New install**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kps prometheus-community/kube-prometheus-stack -n monitoring -f manifests/thanos-values.yaml
```

**3.1b Existing install** – keep current settings and chart version

```bash
helm list -n monitoring                                     # note release name + chart version
helm get values <release> -n monitoring > current-values.yaml
helm upgrade <release> prometheus-community/kube-prometheus-stack -n monitoring \
  --version <same chart version> -f current-values.yaml -f manifests/thanos-values.yaml
```

Why both files: `helm upgrade` rebuilds the config from chart defaults plus the files you pass. `current-values.yaml` keeps every existing setting; `thanos-values.yaml` (last, so it wins) adds the sidecar.

In production, keep the current `retention` at first; reduce it only after Step 6 passes.

**3.2 Check**

```bash
kubectl get pods -n monitoring | grep prometheus-
kubectl get svc -n monitoring | grep thanos-discovery
kubectl logs <prometheus-pod> -n monitoring -c thanos-sidecar | tail -10
```

✅ The Prometheus pod is `3/3 Running`, a `...-thanos-discovery` Service exists, and the sidecar log shows `starting sidecar` with no S3 errors.

## Step 4 – Thanos Store Gateway and Thanos Query

- [`manifests/thanos-store.yaml`](manifests/thanos-store.yaml) – Store Gateway (reads old data from S3) + headless Service
- [`manifests/thanos-query.yaml`](manifests/thanos-query.yaml) – Query (asks sidecar + Store Gateway) + Service

The first `--endpoint` in `thanos-query.yaml` must match the `...-thanos-discovery` Service name from Step 3.2.

Why Services: pod IPs change on every restart. The headless `thanos-store` Service lets Query find Store pods by DNS (`dnssrv+_grpc._tcp...`); the `thanos-query` Service gives Grafana a fixed URL.

```bash
kubectl apply -f manifests/thanos-store.yaml -f manifests/thanos-query.yaml
kubectl get pods -n monitoring | grep thanos
```

✅ Both pods are `1/1 Running`. Open `http://<node-ip>:30909` → **Endpoints**: **Sidecar UP** and **Store UP**.

**Production:** also apply [`manifests/thanos-compactor.yaml`](manifests/thanos-compactor.yaml) (exactly one replica). Set the three `--retention.resolution-*` values to the agreed S3 retention.

## Step 5 – Grafana uses Thanos for all dashboards

Dashboards follow Grafana's **default** data source, so making Thanos the default lets every dashboard show both recent and old data.

**5.1 Open Grafana** (lab: NodePort 30300)

```bash
kubectl patch svc kps-grafana -n monitoring \
  -p '{"spec":{"type":"NodePort","ports":[{"name":"http-web","port":80,"targetPort":3000,"nodePort":30300}]}}'
```

Browse to `http://<node-ip>:30300` and log in as `admin`. Password:

```bash
kubectl get secret kps-grafana -n monitoring -o jsonpath='{.data.admin-password}' | base64 -d
```

**5.2 Add the Thanos data source**

1. **Connections → Data sources → Add new data source → Prometheus**
2. **Name:** `Thanos` (type it; otherwise Grafana names it `prometheus-1`)
3. **Prometheus server URL:** `http://thanos-query.monitoring.svc:9090`
4. Turn on **Default**
5. **Save & test** → ✅ "Successfully queried the Prometheus API"

**5.3 Use it in dashboards**

- Open any dashboard (for example **Node Exporter / Nodes**). At the top, set **Data source = Thanos** if it isn't already.
- Set the time range wider than the local retention (lab: **Last 12 hours**; production: **Last 90 days**). The older part now comes from S3.

## Step 6 – Verify

The first upload happens about **2 hours** after Prometheus starts. The final proof needs data older than the local retention.

| # | Check | Command / where | Expected |
| --- | --- | --- | --- |
| 1 | Sidecar uploaded blocks | `kubectl logs <prometheus-pod> -n monitoring -c thanos-sidecar \| grep -i "upload new block"` | One line per 2h block |
| 2 | Files are in S3 | `echo "s3.bucket.list" \| weed shell` (lab) | `size` greater than 0 |
| 3 | Thanos sees S3 data | `http://<node-ip>:30909` → Endpoints | **Store** row shows Min/Max Time |
| 4 | Prometheus deleted old data | `kubectl exec <prometheus-pod> -n monitoring -c prometheus -- ls /prometheus` | Only the newest block folders |
| 5 | Old data visible in Grafana | Explore → **Split**: left **Prometheus**, right **Thanos**, same query, range wider than retention | Left starts at the retention edge; right shows the full history |

A clear query for check 5 (node memory used, %):

```
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

Prometheus deletes data in whole 2-hour blocks, so with `retention: 6h` it keeps between 6 and 8 hours. That is normal.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Pods evicted, new pods `Pending`, node shows `DiskPressure True` | Worker disk (15 GB) filled up while images downloaded | Grow the disk. On Ubuntu with free space in the volume group: `lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv`. Wait about 5 minutes, then delete the failed pods |
| `thanos-store` in `CrashLoopBackOff`, log shows `dial tcp <S3-IP>:8333: i/o timeout` | Pods cannot reach S3, but the node can. The pod CIDR (`192.168.0.0/16`) overlapped the LAN, so Calico did not NAT the traffic and S3 replies went to the router | Lab: on the S3 VM, add a route for each node's pod range via that node's IP (`nmcli connection modify ens18 +ipv4.routes "<pod-range> <node-ip>"`). Proper fix: a pod CIDR that does not overlap any network pods must reach |
| Store Gateway uses too much memory | Default index cache is 250 MB | `--index-cache-size=64MB` and a 512 Mi limit |
| No `upload new block` in the sidecar log | Too early: the first block is written about 2h after start | Wait, then check again |
| Grafana data source called `prometheus-1` | Name field left empty when adding it | Rename it to `Thanos` under Connections → Data sources |
| Thanos Query web page blank | Browser used `https` | Use `http://<node-ip>:30909` |
| MinIO download returns `410` | Free MinIO binaries are no longer published | Use SeaweedFS (lab) or company S3 (Ceph RGW / NetApp) |

## Production notes

- **Order of changes:** add the sidecar with the current retention, prove uploads and old-data queries (Step 6), then lower the retention (for example to `30d`).
- **Thanos Compactor:** run exactly one. It merges and downsamples blocks and deletes data past the S3 retention. Without it the bucket grows forever.
- **Availability:** run 2 replicas of Thanos Query; give the Store Gateway a PVC instead of `emptyDir`; use `ClusterIP` instead of NodePort.
- **HTTPS S3:** set `insecure: false`; if the endpoint uses an internal CA, add the CA to `objstore.yml` (`http_config.tls_config.ca_file`) and mount it.
- **Air-gapped clusters:** pull the Thanos and Prometheus images into the internal registry first and point the YAML/Helm values to it.
- **Secrets:** never commit real S3 keys to Git; the values in `manifests/objstore.yml` are lab examples only.
- **Rollback:** `helm upgrade` with only `current-values.yaml` removes the sidecar; Prometheus works as before, and data already in S3 stays.
- **Change control:** Step 3 restarts Prometheus once (1–2 minutes without metrics). Raise a change request.
