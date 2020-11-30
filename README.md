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

# 9. Security Contexts (SCC) #
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
