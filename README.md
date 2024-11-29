# skupper-example-cockroachdb

Simple example of a cockroachdb cluster over two different kubernetes
clusters using skupper. This is based on the example from
https://github.com/cockroachdb/cockroach/tree/master/cloud/kubernetes

# Setup

You need two kubernetes clusters, one of which must be reachable from
the other. These instructions assume a separate kube config file for
each cluster. The example yaml uses namespace `east` on the first
cluster and namespace `west` on the second. You can edit these files
to change the namespaces as desired.

It is also assumed that the skupper controller is installed on each of
these clusters.

First we apply the yaml for each cluster:

```kubectl --kubeconfig=first.cfg apply -f ./east.yaml```

and

```kubectl --kubeconfig=second.cfg apply -f ./west.yaml```

Then, once the AccessGrant on east is ready, we need to link the two sites together:

```
kubectl --kubeconfig=first.cfg -n east wait --for=condition=ready accessgrant/east-grant --timeout 5m && kubectl --kubeconfig=first.cfg -n east get accessgrant east-grant -o go-template-file=accesstoken.template | kubectl  --kubeconfig=second.cfg -n west apply -f -
```

Then once the cockroachdb pods in east are running, we initiliase the cluster:

```
kubectl --kubeconfig=first.cfg -n east wait statefulset/cockroachdb-g1 --for=jsonpath='{.status.currentReplicas}'=3 --timeout 5m && kubectl --kubeconfig=first.cfg -n east apply -f ./cluster-init-g1.yaml
```

Once everything has initialised you can view the console by port forwarding on the first cluster:

```kubectl --kubeconfig=first.cfg -n east port-forward cockroachdb-g1-0 8080```

and then accessing http://localhost:8080 with your browser to verify the cockroachdb cluster now has 6 members.

To test you can run a job that will populate records into the `kv` table (`test` database):

```kubectl --kubeconfig=first.cfg -n east create job loadgen-1-minute --image=cockroachdb/loadgen-kv:0.1 -- /kv --duration=1m postgres://root@cockroachdb-public:26257/kv?sslmode=disable```

You can then verify records have been inserted with the following:

```kubectl --kubeconfig=first.cfg -n east run cockroachdb -it --image=cockroachdb/cockroach --rm --restart=Never -- sql --insecure --host=cockroachdb-internal-g1 -e 'select count(*) from test.kv'```
