NEPHIO Scale UPF md komendy:

kubectl config use edge01-admin@edge01
  
  
upf_pod_id=$(kubectl get pods -n free5gc-upf | grep upf | head -1 | cut -d ' ' -f 1)
# Upf deploy
kubectl config use kind-kind

#Take package Downstream target name from packageVariant

upf_packagevariant=$(kubectl get packagevariant edge-free5gc-upf-edge01-free5gc-upf -o jsonpath='{.status.downstreamTargets[0].name}')

package_dir="upf-scale-package"

ws="${cluster}-upf-scaling"
upf_package_revision=$(kpt alpha rpkg copy -n default $upf_packagevariant --workspace $ws | cut -d ' ' -f 1)
# nowy rpkg, nazwa WORKSPACENAME upf-scale-package
kpt alpha rpkg get

# pull to /tmp/upf-scale-package
package_location=/tmp/$ws
kpt alpha rpkg pull -n default "$upf_package_revision" $package_location

### Visualize the difference in Uplink and Downlink throughput using kpt diff 
kpt pkg diff $package_location | grep linkThroughput

# Change capacity 

new_capacity_value=10

kpt fn eval --image gcr.io/kpt-fn/search-replace:v0.2.0 $package_location -- by-path='spec.maxUplinkThroughput' by-file-path='**/capacity.yaml' put-value=$new_capacity_value

kpt fn eval --image gcr.io/kpt-fn/search-replace:v0.2.0 $package_location -- by-path='spec.maxDownlinkThroughput' by-file-path='**/capacity.yaml' put-value=$new_capacity_value


# Package life cycle -> (Push) Draft -> Proposed -> Publish

kpt alpha rpkg push -n default "$upf_pkg_rev" $ws
# kubectl wait?
kpt alpha rpkg propose -n default "$upf_pkg_rev"

kpt alpha rpkg approve -n default "$upf_pkg_rev"








---------------- EXPECTED OUTPUTS ----------------------------

GET PKG

ubuntu@nephio-r1:/tmp $ kpt alpha rpkg get
NAME                                                               PACKAGE                              WORKSPACENAME          REVISION   LATEST   LIFECYCLE   REPOSITORY
edge01-e72d245b864db0fd234d9b4ead2f96edcf6bb3e4                    free5gc-operator                     packagevariant-1       main       false    Published   edge01
edge01-7c9bf9f43768ecd2b45a8be84698763cdd2593b6                    free5gc-operator                     packagevariant-1       v1         true     Published   edge01
edge01-40c616e5d87053350473d3ffa1387a9a534f8f42                    free5gc-upf                          upf-scale-package                 false    Draft       edge01
.
.
.




SCALE DIFF

ubuntu@nephio-r1:/tmp/upf-scale-package$ kpt pkg diff $package_location | grep linkThroughput
From https://github.com/nephio-project/free5gc-packages
 * tag               pkg-example-upf-bp/v3 -> FETCH_HEAD
Adding package "pkg-example-upf-bp".
<   maxUplinkThroughput: 10G
<   maxDownlinkThroughput: 10
>   maxUplinkThroughput: 5G
>   maxDownlinkThroughput: 5G