# Supply Chain Choreographer Templates for Tanzu Application Platform


## How to install supply chains and templates

Add the Package Repository:

```
NAMESPACE=<namespace to install the packages>

tanzu package repository add scc-repository -n ${NAMESPACE} --url ghcr.io/making/scc-repo:0.0.1
```

List the available packages to confirm the addition:

```
$ tanzu package available list -n ${NAMESPACE} 
  NAME                                               DISPLAY-NAME                                                                       SHORT-DESCRIPTION                                                                                           LATEST-VERSION         
  ootb-supply-chain-testing-with-cache.pkg.maki.lol  Out of The Box Supply Chain with Testing and Cache for Tanzu Application Platform  Out of The Box Supply Chain with Testing and Cache.                                                         0.0.1                  
  scc-templates.pkg.maki.lol                         Out of The Box Templates for Tanzu Application Platform                            Out of The Box Templates.                                                                                   0.0.1                  
```


Install new templates:

```yaml
# Note that the service account must have permissions to operate CRUD to pvc
cat <<EOF > scc-templates-values.yaml
service_account: default
cache_size: 1Gi
EOF

tanzu package install scc-templates -n ${NAMESPACE} -p scc-templates.pkg.maki.lol -v 0.0.1 -f scc-templates-values.yaml
```

Install new supply chains:

```yaml
cat <<EOF > ootb-supply-chain-testing-with-cache-values.yaml
registry:
  server: ghcr.io
  repository: making
service_account: default
EOF

tanzu package install ootb-supply-chain-testing-with-cache -n ${NAMESPACE} -p ootb-supply-chain-testing-with-cache.pkg.maki.lol -v 0.0.1 -f ootb-supply-chain-testing-with-cache-values.yaml
```

You'll see new resources are ready as follows:

```
$ kubectl get clustersupplychain,clustertemplate,clustersourcetemplate,clusterruntemplate
NAME                                                         READY   REASON   AGE
clustersupplychain.carto.run/basic-image-to-url              True    Ready    34d
clustersupplychain.carto.run/source-test-with-cache-to-url   True    Ready    35m <<--- 🆕
clustersupplychain.carto.run/source-to-url                   True    Ready    34d

NAME                                                       AGE
clustertemplate.carto.run/config-writer-template           34d
clustertemplate.carto.run/deliverable-template             34d
clustertemplate.carto.run/tekton-pipeline-cache-template   35m <<--- 🆕

NAME                                                          AGE
clustersourcetemplate.carto.run/delivery-source-template      34d
clustersourcetemplate.carto.run/source-scanner-template       34d
clustersourcetemplate.carto.run/source-template               34d
clustersourcetemplate.carto.run/testing-pipeline              34d
clustersourcetemplate.carto.run/testing-pipeline-with-cache   35m <<--- 🆕

NAME                                                                AGE
clusterruntemplate.carto.run/tekton-source-pipelinerun              34d
clusterruntemplate.carto.run/tekton-source-pipelinerun-with-cache   35m <<--- 🆕
clusterruntemplate.carto.run/tekton-taskrun                         34d
```

## Out of The Box Supply Chain with Testing and Cache

For example, create the following pipeline that uses maven as a build tool and caches the dependencies

```yaml
NAMESPACE=<namespace for a workload>

cat <<'EOF' > maven-test-pipeline.yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: maven-test-pipeline
  labels:
    apps.tanzu.vmware.com/pipeline: test
    apps.jaguchi.maki.lol/has-cache: "true"
spec:
  params:
  - name: source-url
  - name: source-revision
  workspaces:
  - name: pipeline-cache
  tasks:
  - name: test
    params:
    - name: source-url
      value: $(params.source-url)
    - name: source-revision
      value: $(params.source-revision)
    workspaces:
    - workspace: pipeline-cache
      name: pipeline-cache
    taskSpec:
      params:
      - name: source-url
      - name: source-revision
      workspaces:
      - name: pipeline-cache
      steps:
      - name: test
        image: eclipse-temurin:17
        script: |-
          set -ex
          rm -rf ~/.m2
          ln -fs $(workspaces.pipeline-cache.path) ~/.m2
          cd `mktemp -d`
          curl -s $(params.source-url) | tar -xzf -
          ./mvnw clean test -V --no-transfer-progress
EOF
kubectl apply -f maven-test-pipeline.yaml -n ${NAMESPACE}
```

Create a maven based workload

```
tanzu apps workload apply hello-servlet \             
  --app hello-servlet \
  --git-repo https://github.com/making/hello-servlet \
  --git-branch master \
  --type web \
  --label apps.tanzu.vmware.com/has-tests=true \
  --label apps.jaguchi.maki.lol/has-cache=true \
  -n ${NAMESPACE} -y
```

Use [`stern`](https://github.com/stern/stern) to watch the logs

```
stern -n ${NAMESPACE} hello-servlet
```


First time, you'll see the test took about 1.5 min.

```
hello-servlet-m8mn4-test-pod step-test [INFO] ------------------------------------------------------------------------
hello-servlet-m8mn4-test-pod step-test [INFO] BUILD SUCCESS
hello-servlet-m8mn4-test-pod step-test [INFO] ------------------------------------------------------------------------
hello-servlet-m8mn4-test-pod step-test [INFO] Total time:  01:29 min
hello-servlet-m8mn4-test-pod step-test [INFO] Finished at: 2022-06-14T18:23:09Z
hello-servlet-m8mn4-test-pod step-test [INFO] ------------------------------------------------------------------------
```

Then delete the `PipelineRun` resource as follows:

```
kubectl delete pipelinerun  -l app.kubernetes.io/part-of=hello-servlet -n ${NAMESPACE} 
```

After a while you will see the test run again and complete in **a few seconds**.

```
hello-servlet-g5wlm-test-pod step-test [INFO] ------------------------------------------------------------------------
hello-servlet-g5wlm-test-pod step-test [INFO] BUILD SUCCESS
hello-servlet-g5wlm-test-pod step-test [INFO] ------------------------------------------------------------------------
hello-servlet-g5wlm-test-pod step-test [INFO] Total time:  2.623 s
hello-servlet-g5wlm-test-pod step-test [INFO] Finished at: 2022-06-14T18:43:27Z
hello-servlet-g5wlm-test-pod step-test [INFO] ------------------------------------------------------------------------
```



## How to create the carvel package

```
VERSION=0.0.1
kbld -f bundle/ootb-supply-chain-testing-with-cache/config --imgpkg-lock-output bundle/ootb-supply-chain-testing-with-cache/.imgpkg/images.yml
imgpkg push -b ghcr.io/making/ootb-supply-chain-testing-with-cache-bundle:${VERSION} -f bundle/ootb-supply-chain-testing-with-cache
ytt -f bundle/ootb-supply-chain-testing-with-cache/values.yaml --data-values-schema-inspect -o openapi-v3 > /tmp/ootb-supply-chain-testing-with-cache-schema-openapi.yml
ytt -f template/ootb-supply-chain-testing-with-cache.yaml --data-value-file openapi=/tmp/ootb-supply-chain-testing-with-cache-schema-openapi.yml -v version=${VERSION} > repo/packages/ootb-supply-chain-testing-with-cache.pkg.maki.lol/${VERSION}.yaml

kbld -f bundle/scc-templates/config --imgpkg-lock-output bundle/scc-templates/.imgpkg/images.yml
imgpkg push -b ghcr.io/making/scc-templates-bundle:${VERSION} -f bundle/scc-templates
ytt -f bundle/scc-templates/values.yaml --data-values-schema-inspect -o openapi-v3 > /tmp/scc-templates-schema-openapi.yml
ytt -f template/scc-templates.yaml --data-value-file openapi=/tmp/scc-templates-schema-openapi.yml -v version=${VERSION} > repo/packages/scc-templates.pkg.maki.lol/${VERSION}.yaml


kbld -f repo/packages --imgpkg-lock-output repo/.imgpkg/images.yml
imgpkg push -b ghcr.io/making/scc-repo:${VERSION} -f repo
```
