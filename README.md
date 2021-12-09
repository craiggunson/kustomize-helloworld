


# Demo: hello world with variants

This has been modified from the[original](https://github.com/kubernetes-sigs/kustomize/tree/master/examples/helloWorld) to use the built in `kubectl kustomize` or `kubectl apply -k` commands, also the glossary moved to [here](https://kubectl.docs.kubernetes.io/references/kustomize/glossary/).  


Steps:

 1. Clone an existing configuration as a [base].
 1. Customize it.
 1. Create two different [overlays] (_staging_ and _production_)
    from the customized base.
 1. Run kustomize and kubectl to deploy staging and production.

First define a place to work:


 ```
 DEMO_HOME=~/hello
 ```

## Establish the base

Let's run the [hello] service.

To use [overlays] to create [variants], we must
first establish a common [base].

To keep this document shorter, the base resources are
off in a supplemental data directory rather than
declared here as HERE documents.  Download them:

<!-- @downloadBase @testAgainstLatestRelease -->
```
BASE=$DEMO_HOME/base
mkdir -p $BASE

curl -s -o "$BASE/#1.yaml" "https://raw.githubusercontent.com\
/kubernetes-sigs/kustomize\
/master/examples/helloWorld\
/{configMap,deployment,kustomization,service}.yaml"
```

Look at the directory:

<!-- @runTree -->
```
tree $DEMO_HOME
```

Expect something like:

```
/tmp/tmp.IyYQQlHaJP
└── base
    ├── configMap.yaml
    ├── deployment.yaml
    ├── kustomization.yaml
    └── service.yaml
```


One could immediately apply these resources to a
cluster.

```
kubectl apply -k $DEMO_HOME/base
```

to instantiate the _hello_ service.  `kubectl`
would only recognize the resource files.

### The Base Kustomization

The `base` directory has a [kustomization] file:

<!-- @showKustomization @testAgainstLatestRelease -->
```
more $BASE/kustomization.yaml
```

Optionally, run `kustomize` on the base to emit
customized resources to `stdout`:

<!-- @buildBase @testAgainstLatestRelease -->
```
kustomize build $BASE
or
kubectl kustomize $BASE
```

### Customize the base

A first customization step could be to change the _app
label_ applied to all resources:

<!-- @addLabel @testAgainstLatestRelease -->
```
sed -i.bak 's/app: hello/app: my-hello/' $BASE/kustomization.yaml
```

See the effect:
<!-- @checkLabel @testAgainstLatestRelease -->
```
kustomize build $BASE | grep -C 3 app:
or
kubectl kustomize $BASE | grep -C 3 app:
```

## Create Overlays

Create a _staging_ and _production_ [overlay]:

 * _Staging_ enables a risky feature not enabled in production.
 * _Production_ has a higher replica count.
 * Web server greetings from these cluster
   [variants] will differ from each other.

<!-- @overlayDirectories @testAgainstLatestRelease -->
```
OVERLAYS=$DEMO_HOME/overlays
mkdir -p $OVERLAYS/staging
mkdir -p $OVERLAYS/production
```

#### Staging Kustomization

In the `staging` directory, make a kustomization
defining a new name prefix, and some different labels.

<!-- @makeStagingKustomization @testAgainstLatestRelease -->
```
cat <<'EOF' >$OVERLAYS/staging/kustomization.yaml
namePrefix: staging-
commonLabels:
  variant: staging
  org: acmeCorporation
commonAnnotations:
  note: Hello, I am staging!
resources:
- ../../base
patchesStrategicMerge:
- map.yaml
EOF
```

#### Staging Patch

Add a configMap customization to change the server
greeting from _Good Morning!_ to _Have a pineapple!_

Also, enable the _risky_ flag.

<!-- @stagingMap @testAgainstLatestRelease -->
```
cat <<EOF >$OVERLAYS/staging/map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: the-map
data:
  altGreeting: "Have a pineapple!"
  enableRisky: "true"
EOF
```

#### Production Kustomization

In the production directory, make a kustomization
with a different name prefix and labels.

<!-- @makeProductionKustomization @testAgainstLatestRelease -->
```
cat <<EOF >$OVERLAYS/production/kustomization.yaml
namePrefix: production-
commonLabels:
  variant: production
  org: acmeCorporation
commonAnnotations:
  note: Hello, I am production!
resources:
- ../../base
patchesStrategicMerge:
- deployment.yaml
EOF
```


#### Production Patch

Make a production patch that increases the replica
count (because production takes more traffic).

<!-- @productionDeployment @testAgainstLatestRelease -->
```
cat <<EOF >$OVERLAYS/production/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: the-deployment
spec:
  replicas: 10
EOF
```

## Compare overlays


`DEMO_HOME` now contains:

 - a _base_ directory - a slightly customized clone
   of the original configuration, and

 - an _overlays_ directory, containing the kustomizations
   and patches required to create distinct _staging_
   and _production_ [variants] in a cluster.

Review the directory structure and differences:

<!-- @listFiles -->
```
tree $DEMO_HOME
```

Expecting something like:

```
/tmp/tmp.IyYQQlHaJP1
├── base
│   ├── configMap.yaml
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
└── overlays
    ├── production
    │   ├── deployment.yaml
    │   └── kustomization.yaml
    └── staging
        ├── kustomization.yaml
        └── map.yaml
```

Compare the output directly
to see how _staging_ and _production_ differ:


```
colordiff --side-by-side   <(kubectl kustomize ./overlays/staging)   <(kubectl kustomize ./overlays/production) | more
```

The first part of the difference output should look
something like
```diff
apiVersion: v1                                                  apiVersion: v1
data:                                                           data:
+  altGreeting: Have a pineapple!                              |   altGreeting: Good Morning!
+  enableRisky: "true"                                         |   enableRisky: "false"
kind: ConfigMap                                                 kind: ConfigMap
metadata:                                                       metadata:
  annotations:                                                    annotations:
+    note: Hello, I am staging!                                |     note: Hello, I am production!
  labels:                                                         labels:
    app: my-hello                                                   app: my-hello
    org: acmeCorporation                                            org: acmeCorporation
+    variant: staging                                          |     variant: production
+  name: staging-the-map                                       |   name: production-the-map
```
## Deploy

The individual resource sets are:

<!-- @buildStaging @testAgainstLatestRelease -->
```
kubectl kustomize $OVERLAYS/staging
```

<!-- @buildProduction @testAgainstLatestRelease -->
```
kubectl kustomize $OVERLAYS/production
```

To deploy, pipe the above commands to kubectl apply:

```
kustomize build $OVERLAYS/staging | kubectl apply -f -
or 
kubectl apply -k $OVERLAYS/staging

```
  
```
kustomize build $OVERLAYS/production | kubectl apply -f -
or
kubectl apply -k $OVERLAYS/production

```
