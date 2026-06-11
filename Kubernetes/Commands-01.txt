kubectl config get-contexts
kubectl config use-context ai-k8s-cluster1

kubectl config rename-context ai-k8s-cluster1 cluster1
kubectl config rename-context kubernetes-c2-context cluster2


export KUBECONFIG=~/.kube/config-bkp:~/.kube/c2-kubeconfig
kubectl config view --flatten > ~/.kube/merged-config
cp ~/.kube/merged-config ~/.kube/config


