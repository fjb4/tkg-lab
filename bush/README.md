# Install Harbor on TKG with SSL

tmc cluster create --account-name bush --ssh-key-name tmc-key-pair --region us-east-2 --worker-node-count 2 --instance-type m5.large --group bush --name bush-tkg-1

tmc cluster provisionedcluster kubeconfig get bush-tkg-1
kubectl konfig import --save <filename>

kubectl create clusterrolebinding privileged-role-binding --clusterrole=vmware-system-tmc-psp-privileged --group=system:authenticated

Install metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml

kapp deploy -a cert-manager -n tanzu-kapp -f tkg-extensions/cert-manager/

kapp deploy -a contour -n tanzu-kapp -f tkg-extensions/ingress/contour/aws
kubectl get pod,svc -n tanzu-system-ingress
aws elb describe-load-balancers

helm upgrade --install external-dns bitnami/external-dns -n tanzu-system-ingress -f ./external-dns-values.yaml 

## Contour
kubectl annotate service envoy "external-dns.alpha.kubernetes.io/hostname=tkg.jb-cloud.com." -n tanzu-system-ingress --overwrite

kubectl create secret generic prod-route53-credentials-secret --from-literal=secret-access-key='YfWEOaQfFfUehOFGmjkx/wVMftYn3r3u0HCqVSxh' -n cert-manager

kubectl apply -f cluster-issuer.yaml  

kubectl get clusterissuer letsencrypt-contour-cluster-issuer -o yaml

## Harbor
helm template harbor harbor/harbor -f harbor-values.yaml -n harbor > harbor-helm-manifest.yaml
kapp deploy -a harbor -n tanzu-kapp --into-ns harbor -f harbor-certs.yaml -f harbor-helm-manifest.yaml

## Kubeapps
helm install kubeapps --namespace kubeapps bitnami/kubeapps -n kubeapps --set ingress.enabled=true --set ingress.hostname=kubeapps.tkg.jb-cloud.com

kubectl create serviceaccount kubeapps-operator -n kubeapps
kubectl create clusterrolebinding kubeapps-operator --clusterrole=cluster-admin --serviceaccount=kubeapps:kubeapps-operator

## Gitlab

kubectl create ns gitlab
kubectl apply -f gitlab-certs.yaml  

helm install gitlab gitlab/gitlab -n gitlab --timeout 600s -f ./gitlab-values.yaml

### Retrieve API token
kubectl get secret -n kubeapps $(kubectl get serviceaccount kubeapps-operator -n kubeapps -o jsonpath='{range .secrets[*]}{.name}{"\n"}{end}' | grep kubeapps-operator-token) -o jsonpath='{.data.token}' -o go-template='{{.data.token | base64decode}}' && echo

# Install TAS4K8s

https://docs-pcf-staging.cfapps.io/tas-kubernetes/0-n/index.html
https://network.pivotal.io/products/tas-for-kubernetes/

tmc cluster create --account-name bush --ssh-key-name tmc-key-pair --region us-east-2 --worker-node-count 4 --instance-type m5.xlarge --group bush --name bush-tas-1

tmc cluster provisionedcluster kubeconfig get bush-tas-1

kubectl create clusterrolebinding privileged-role-binding --clusterrole=vmware-system-tmc-psp-privileged --group=system:authenticated

kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml

Install Tanzu Postgres
http://postgres-kubernetes.docs.pivotal.io/0-4/installing.html
https://network.pivotal.io/products/pivotal-postgres-for-kubernetes/





kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.0/cert-manager.yaml

pivnet download-product-files --product-slug='pivotal-postgres-for-kubernetes' --release-version='1.0.0-c1' --product-file-id=806931

tar xzfv postgres-for-kubernetes-v1.0.0-c1.tar.gz

docker load -i ./images/postgres-instance
docker load -i ./images/postgres-operator
docker images "postgres-*"

docker tag d985f1ff8369 harbor.jb-cloud.com/tanzu-postgres/postgres-operator:v1.0.0-c1
docker push harbor.jb-cloud.com/tanzu-postgres/postgres-operator:v1.0.0-c1

docker tag 2ab76c8f1b4a harbor.jb-cloud.com/tanzu-postgres/postgres-instance:v1.0.0-c1
docker push harbor.jb-cloud.com/tanzu-postgres/postgres-instance:v1.0.0-c1

kubectl create secret docker-registry regsecret \
    --docker-server=harbor.jb-cloud.com \
    --docker-username=admin \
    --docker-password="Pass@word1"

helm install postgres-operator operator/

helm install postgres-operator -f operator/values-overrides.yaml operator/

kubectl create -f pg-small-instance.yaml

Tanzu Application Service for Kubernetes
https://docs-pcf-staging.cfapps.io/tas-kubernetes/0-n/index.html
https://network.pivotal.io/products/tas-for-kubernetes#/releases/759070

pivnet download-product-files --product-slug='pivotal-postgres-for-kubernetes' --release-version='1.0.0-c1' --product-file-id=806931

