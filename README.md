# skupper-example-cockroachdb

Simple example of a cockroachdb cluster over three different kubernetes
clusters using skupper. This is based on the example from
https://github.com/cockroachdb/cockroach/tree/master/cloud/kubernetes

# Setup

You need three kubernetes clusters, the first of of which must be
accessible by the others. Note: at present the namespaces on each of
these clusters must have the same name.

To begin with you need to replace ${NAMESPACE} in the yaml files for
the three statefulsets to be the value of your current namespace.

In the first kubernetes cluster:

1. create the cockroachdb statefulset

If namespace has already been changed:

```kubectl apply -f ./cockroachdb-statefulset-g1.yaml```

Or to dynamically change it:

```NAMESPACE=$(kubens --current) envsubst < cockroachdb-statefulset-g1.yaml | kubectl apply -f -```

2. once those pods are running, initialise the cluster

```kubectl apply -f ./cluster-init-g1.yaml```

3. initialise skupper

```skupper init```

4. create a connection tokens with which the second kubernetes cluster
will connect to this first one

```skupper token create token-1.yaml```

5. create a connection tokens with which the third kubernetes cluster
will connect to this first one

```skupper token create token-2.yaml```

Now in the second kubernetes cluster:

6. initialise skupper

```skupper init```

7. connect to the first site using the connection token created in step 4

```skupper link create token-1.yaml```

8. create another cockroachdb statefulset in the second cluster

```kubectl apply -f ./cockroachdb-statefulset-g2.yaml```

Or:

```NAMESPACE=$(kubens --current) envsubst < cockroachdb-statefulset-g2.yaml | kubectl apply -f -```

Now in the third kubernetes cluster:

9. initialise skupper

```skupper init```

10. connect to the first site using the connection token created in step 5

```skupper link create token-2.yaml```

11. create another cockroachdb statefulset in the third cluster

```kubectl apply -f ./cockroachdb-statefulset-g3.yaml```

Or:

```NAMESPACE=$(kubens --current) envsubst < cockroachdb-statefulset-g3.yaml | kubectl apply -f -```

Once everything has initialised you can verify the cockroachdb cluster now has 9 members by port forwarding on the first cluster:

```kubectl port-forward cockroachdb-g1-0 8080```

and then accessing http://localhost:8080 with your browser

12. populate a sample database

```kubectl create job loadgen-1-minute --image=cockroachdb/loadgen-kv:0.1 -- /kv --duration=1m postgres://root@cockroachdb-public:26257/kv?sslmode=disable```

The job above will run for 1 minute and will populate records into the `kv` table (`test` database).

13. verify records have been inserted

```kubectl run cockroachdb -it --image=cockroachdb/cockroach --rm --restart=Never -- sql --insecure --host=cockroachdb-internal-g1 -e 'select count(*) from test.kv'```
```kubectl run cockroachdb -it --image=cockroachdb/cockroach --rm --restart=Never -- sql --insecure --host=cockroachdb-internal-g2 -e 'select count(*) from test.kv'```
```kubectl run cockroachdb -it --image=cockroachdb/cockroach --rm --restart=Never -- sql --insecure --host=cockroachdb-internal-g3 -e 'select count(*) from test.kv'```
