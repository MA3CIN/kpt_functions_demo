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
kpt alpha rpkg pull -n default "$upf_package_revision" package_location

### Visualize the difference in Uplink and Downlink throughput using kpt diff 
kpt pkg diff . | grep linkThroughput

# Change capacity 

new_capacity_value=10

kpt fn eval --image gcr.io/kpt-fn/search-replace:v0.2.0 $package_location -- by-path='spec.maxUplinkThroughput' by-file-path='**/capacity.yaml' put-value=$new_capacity_value

kpt fn eval --image gcr.io/kpt-fn/search-replace:v0.2.0 $package_location -- by-path='spec.maxDownlinkThroughput' by-file-path='**/capacity.yaml' put-value=$new_capacity_value


# Package life cycle -> (Push) Draft -> Proposed -> Publish

kpt alpha rpkg push -n default "$upf_pkg_rev" $ws
# kubectl wait?
kpt alpha rpkg propose -n default "$upf_pkg_rev"

kpt alpha rpkg approve -n default "$upf_pkg_rev"
