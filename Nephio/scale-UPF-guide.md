## Step 9: Change the Capacities of the UPF and SMF NFs

In this step, you will change the capacity requirements for the UPF and SMF, and
see how the operator reconfigures the Kubernetes resources used by the network
functions.

The capacity requirements are captured in a custom resource (capacity.yaml) within the deployed
package.

Start by saving the package Variant downstream target name to a variable.

```
kubectl config use kind-kind
upf_packagevariant=$(kubectl get packagevariant edge-free5gc-upf-edge01-free5gc-upf -o jsonpath='{.status.downstreamTargets[0].name}')

```
Next create a new package revision from the existing UPF package.

```
ws="upf-scale-package"
upf_package_revision=$(kpt alpha rpkg copy -n default $upf_packagevariant --workspace $ws | cut -d ' ' -f 1)
```

Pull the package to a local directory of your choice. The contents of the package will be mutated using kpt functions to adjust the UPF configuration.
```
package_location=/tmp/$ws
kpt alpha rpkg pull -n default "$upf_package_revision" $package_location
```

Apply the kpt functions to the contents of the kpt package with a new value for the throughputs of your choice.
```
new_capacity_value=10

kpt fn eval --image gcr.io/kpt-fn/search-replace:v0.2.0 $package_location -- by-path='spec.maxUplinkThroughput' by-file-path='**/capacity.yaml' put-value=$new_capacity_value

kpt fn eval --image gcr.io/kpt-fn/search-replace:v0.2.0 $package_location -- by-path='spec.maxDownlinkThroughput' by-file-path='**/capacity.yaml' put-value=$new_capacity_value
```

Observe the changes to the UPF configuration using the kpt pkg diff command.
```
kpt pkg diff $package_location | grep linkThroughput
``` 

<details>
<summary>The output is similar to:</summary>
```
ubuntu@nephio-r1:/tmp/upf-scale-package$ kpt pkg diff $package_location | grep linkThroughput
From https://github.com/nephio-project/free5gc-packages
 * tag               pkg-example-upf-bp/v3 -> FETCH_HEAD
Adding package "pkg-example-upf-bp".
<   maxUplinkThroughput: 10G
<   maxDownlinkThroughput: 10
>   maxUplinkThroughput: 5G
>   maxDownlinkThroughput: 5G
```
</details>

Next, progress through the package lifecycle stages by pushing the changes to the package to its repository, proposing the changes and approving them.

```
kpt alpha rpkg push -n default "$upf_package_revision" $package_location

kpt alpha rpkg propose -n default "$upf_package_revision"

kpt alpha rpkg approve -n default "$upf_package_revision"
```

We can check the current lifecycle stage of a package using the kpt get command.


<details>
<summary>The output is similar to:</summary>
```
ubuntu@nephio-r1:/tmp $ kpt alpha rpkg get
NAME                                                               PACKAGE                              WORKSPACENAME          REVISION   LATEST   LIFECYCLE   REPOSITORY
edge01-e72d245b864db0fd234d9b4ead2f96edcf6bb3e4                    free5gc-operator                     packagevariant-1       main       false    Published   edge01
edge01-7c9bf9f43768ecd2b45a8be84698763cdd2593b6                    free5gc-operator                     packagevariant-1       v1         true     Published   edge01
edge01-40c616e5d87053350473d3ffa1387a9a534f8f42                    free5gc-upf                          upf-scale-package      v2         true     Published   edge01
```
</details>

After the package is approved, check the status of the UPF pod on the edge cluster.

```
kubectl config use edge01-admin@edge01
kubectl get po -n 
```
