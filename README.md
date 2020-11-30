# 1. Labels #
##
oc label node node1 env=dev 

oc label node node2 env=prod

oc label node node1 env=test --overwrite

oc get nodes --show-labels

oc label node node1 env-

# 2. Node Selector #
##
oc patch deployment/myapp --patch '{"spec":{"template":{"spec":{"nodeSelector":{"env":"dev"}}}}}'

oc adm new-project demo --node-selector "env=dev"

oc annotate namespace demo openshift.io/node-selector "env=dev" --overwrite

oc patch namespace demo --patch '{"metadata":{"annotations":{"openshift.io/node-selector": "env=dev"}}}'

# 3. Taints #
##
oc adm taint nodes node1 dedicated=foo:NoSchedule -o json --dry-run=client | jq .spec.taints
```
[
  {
    "effect": "NoSchedule",
    "key": "dedicated",
    "value": "foo"
  }
]
```

oc describe node node1 |grep -A2 Taint
```
Taints:      dedicated=foo:NoSchedule
             test=foo:NoSchedule
             
```
oc adm taint node node1 dedicated-

oc adm taint node node1 test-

# 4. OAuth #
sudo yum install httpd-tools

htpasswd -c -b /tmp/htpass user1 password1

htpasswd -b /tmp/htpass user2 password2

oc get oauth cluster -o yaml > /tmp/oauth.yaml

add httpasswd to oauth.yaml
```
spec:
  identityProviders:
  - name: localusers
    type: HTPasswd
    mappingMethod: claim
    htpasswd:
      fileData:
        name: /tmp/htpass
```
oc replace -f /tmp/oauth.yaml

oc create secret generic htpass-secret --from-file htpasswd=/tmp/htpass -n openshift-config

oc get po -w -n openshift-authentication

Delete user from htpass<br/>
&nbsp;&nbsp;&nbsp;&nbsp;htpasswd -D /tmp/htpass user2

Update the secret with new htpass<br/>
&nbsp;&nbsp;&nbsp;&nbsp;oc set data secret/htpass-secret --from-file htpasswd=/tmp/htpass -n openshift-config

oc delete user user2

oc delete identity localusers:user2

oc delete user --all
oc delete identity --all

# 5. Users, Groups, and Authentication #
oc __adm__ policy add-cluster-role-to-user cluster-admin user-name    #__cluster role__

oc policy add-role-to-user role-name user-name -n project-name        #__namespace level role__

oc get clusterrolebindings -o wide |grep -E "NAME|self-provisioners"

oc describe clusterrolebindings self-provisioners

oc describe clusterrole self-provisioner

Remove self-provisioner role from system such that authenticated users can't create projects<br/>
&nbsp;&nbsp;&nbsp;&nbsp;oc adm policy remove-cluster-role-from-group self-provisioner system:authenticated:oauth

Restore self-provisioners back to cluster as original<br/>
&nbsp;&nbsp;&nbsp;&nbsp;oc adm policy add-cluster-role-to-group --rolebinding-name self-provisioners self-provisioner system:authenticated:oauth

oc adm group new dev-users dev1 dev2

oc adm group new qa-users qa1 qa2

oc policy add-role-to-group __edit__ dev-users -n namespace

oc policy add-role-to-group __view__ qa-users -n namespace

oc policy add-role-to-user admin user1 -n namespace

Get all the rolebindings for the current namespace<br/>
&nbsp;&nbsp;&nbsp;&nbsp;oc get rolebindings -o wide

# 6. Remove kubeadmin from the system #
##
Make sure you have assigned cluster-admin to someone else before doing this! Instead of removing/deleting the user, we will remove the password from the system. <br/>
&nbsp;&nbsp;&nbsp;&nbsp;oc delete secret kubeadmin -n kube-system

# 7. Secrets and ConfigMaps #
##
oc create secret generic secretname --from-literal key1=value1 --from-literal key2=value2

oc set env deployment/hello --from secret/secretname

oc set volume deployment/demo --add --type secret --secret-name demo-secret --mount-path /app-secrets

oc create secret generic mysql --from-literal user=dba --from-literal password=redhat123 --from-literal database=test --from-literal hostname=mysql

oc new-app --name mysql --docker-image registry.access.redhat.com/rhscl/mysql-57-rhel7:5.7-47

oc set env deployment/mysql --prefix MYSQL_ --from secret/mysql

# 8. Secure Routes #
##
```
openssl req -x509 -newkey rsa:2048 -nodes -keyout cert.key -out cert.crt -subj "/C=US/ST=FL/L=Tampa/O=IBM/CN=*.route-hostname"
```
oc create secret tls demo-certs --cert cert.crt --key cert.key

oc set volume deployment/demo --add --type=secret --secret-name demo-tls --mount-path /usr/local/etc/ssl/certs --name tls-mount

oc create route passthrough demo-https --service demo-https --port 8443 --hostname demo-https.apps.ocp4.example.com

oc expose route edge demo-https --service api-frontend --hostname api.apps.acme.com --key cert.key --cert cert.crt

oc extract secrets/router-ca --keys tls.crt -n openshift-ingress-operator --to /tmp/ 

# 9. Security Contexts (SCC) and ServiceAccounts #
##
oc get scc

oc describe scc anyuid

oc create serviceaccount svc-name

oc get po/podname-756ff-9cjbj -o yaml | oc adm policy scc-subject-review -f -

oc adm policy add-scc-to-user anyuid -z svc-name -n namespace

oc set serviceaccount deployment/demo svc-name

```
oc new-app --name gitlab --docker-image quay.io/redhattraiing/gitlab-ce:8.4.3-ce.0
oc get po/gitlab-6c5b5c5d55-gzkct -o yaml | oc adm policy scc-subject-review -f -
oc create sa gitlab-sa
oc adm policy add-scc-to-user anyuid -z gitlab-sa
oc set sa deployment/gitlab gitlab-sa
```

# 10. Limits, Quotas, and LimitRanges #
##
Check the node resources<br/>
`oc describe node node1`

Set resources on a deployment. Request limits are how much each request is allowed, and limit is the max allowed <br/>
`oc set resources deployment hello-world-nginx --requests cpu=10m,memory=20Mi --limits cpu=180m,memory=100Mi`

Quota is project level resources available<br/>
`oc create quota dev-quota --hard services=10,cpu=1300,memory=1.5Gi`

Cluster Quota is resources available across multiple projects<br/>
`oc create clusterquota env-qa --project-annotation-selector.openshift.io/requester=qa --hard pods=12,secrets=20,services=5`

Limit ranges in a yaml file are defined as follows
```
apiVersion: "v1"
kind: "LimitRange"
metadata:
  name: "nms-limits"
spec:
  limits:
    - type: "Pod"
      max:
        cpu: "2"
        memory: "1Gi"
      min:
        cpu: "200m"
        memory: "6Mi"
    - type: "Container"
      max:
        cpu: "2"
        memory: "1Gi"
      min:
        cpu: "100m"
        memory: "4Mi"
      default:			# default that a container can use if not specified in the Pod spec
        cpu: "300m"
        memory: "200Mi"
      defaultRequest:	# default that a container can request if not specified in the Pod spec
        cpu: "200m"
        memory: "100Mi"
```
oc create -f limitranges.yaml -n net-ingress

oc describe limits nms-limits -n net-ingress
```
Name:       nms-limits
Namespace:  net-ingress
Type        Resource  Min   Max  Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---   ---  ---------------  -------------  -----------------------
Pod         cpu       200m  2    -                -              -
Pod         memory    6Mi   1Gi  -                -              -
Container   cpu       100m  2    200m             300m           -
Container   memory    4Mi   1Gi  100Mi            200Mi          -

```
# 11. Scaling and AutoScaler #
##
oc scale --replicas 3 deployment/demo

oc autoscale dc/demo --min 1 --max 10 --cpu-percent 80

oc get hpa

# 12. Image Registry #
##
Built in registry is housed in openshift-image-registry namespace<br/>
`oc -n openshift-image-registry get svc`

cluster wide svc nameing scheme is service-name.namespace.svc<br/>
`image-registry.openshift-image-registry.svc`

using stored images from openshift project
```
oc get images
oc get is -n openshift | grep httpd
```

```
oc new-app --name nms --docker-image image-registry.openshift-image-registry.svc:5000/openshift/httpd:latest
      OR
oc new-app --name nms image-registry.openshift-image-registry.svc:5000/openshift/httpd:latest
      OR
oc new-app --name nms --image image-registry.openshift-image-registry.svc:5000/openshift/httpd:latest
```
Create a service<br/>
`oc expose deployment/nms --port 8080 --target-port 8080`

# 13. Deployment Strategy #
##
Using a docker strategy using local build directory
```
oc new-app --strategy docker --binary --name myapp
oc start-build myapp --from-dir . --follow
oc expose deployment myapp --target-port 8080 --port 80
oc expose svc myapp
```

Using a nodejs builder
```
oc new-app --binary --image-stream nodejs --name nodejs
oc start-build nodejs --from-dir . --follow
oc expose svc/nodejs
```
