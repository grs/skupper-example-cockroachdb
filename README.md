# skupper-example-cockroachdb
Simple example of a cockroachdb cluster over two different kubernetes clusters using skupper

# Setup

You need two kubernetes clusters, the first of of which must be accessible by
the other.

In the first kubernetes cluster:

1. create the cockroachdb statefulset

```kubectl apply -f ./cockroachdb-statefulset-g1.yaml```

2. once those pods are running, initialise the cluster

```kubectl apply -f ./cluster-init-g1.yaml```

3. initialise skupper

```skupper init```

4. create a connection token with which the second kubernetes cluster
will connect to this first one

```skupper connection-token site-one.yaml```

Now in the second kubernetes cluster:

5. initialise skupper

```skupper init```

6. connect the two skupper sites using the connection token created in step 4

```skupper connect site-one.yaml```

7. create another cockroachdb statefulset in the second cluster

```kubectl apply -f ./cockroachdb-statefulset-g2.yaml```

Once everything has initialised you can verify the cockroachdb cluster now has 5 members by port forwarding on the first cluster:

```kubectl port-forward cockroachdb-g1-0 8080```

and then accessing http://localhost:8080 with your browser

