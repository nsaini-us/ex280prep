# 1. Labels #

`oc label node node1 env=dev` <br/>
`oc label node node2 env=prod` <br/>
`oc label node node1 env=test --overwrite` <br/>
`oc get nodes --show-labels` <br/>
`oc label node node1 env-` <br/>

# 2. Node Selector #

`oc patch deployment/myapp \` <br/>
&nbsp;&nbsp;`--patch '{"spec":{"template":{"spec":{"nodeSelector":{"env":"dev"}}}}}'`
        
`oc adm new-project demo --node-selector "env=dev"` <br/>
`oc annotate namespace demo openshift.io/node-selector "env=dev" --overwrite` <br/>
`oc patch namespace demo \` <br/>
&nbsp;&nbsp;`--patch '{"metadata":{"annotations":{"openshift.io/node-selector": "env=dev"}}}'`

# 3. Taints #

`oc adm taint nodes node1 dedicated=foo:NoSchedule -o json --dry-run=client |\` <br/>
&nbsp;&nbsp;`jq .spec.taints`

```
[
  {
    "effect": "NoSchedule",
    "key": "dedicated",
    "value": "foo"
  }
]
```

`oc describe node node1 | grep -A2 Taint`

```
Taints:      dedicated=foo:NoSchedule
             test=foo:NoSchedule
             
```
Delete node taint<br/>
`oc adm taint node node1 dedicated-`

Delete the other key<br/>
`oc adm taint node node1 test-`

# 4. OAuth #

Install htpasswd command line utility<br/>
`sudo yum install httpd-tools`

Create a new file for htpass with users and their passwords

`htpasswd -c -b /tmp/htpass user1 password1` <br/>
`htpasswd -b /tmp/htpass user2 password2`

Copy existing oauth config to a file for editing<br/>
`oc get oauth cluster -o yaml > /tmp/oauth.yaml`

Add httpasswd to oauth.yaml
```
spec:
  identityProviders:
  - name: localusers
    type: HTPasswd
    mappingMethod: claim
    htpasswd:
      fileData:
        name: htpass-secret
```
Create a new secret which will hold the users and password file<br/>

`oc create secret generic htpass-secret \` <br/>
&nbsp;&nbsp;`--from-file htpasswd=/tmp/htpass -n openshift-config`

Now merge/replace existing oauth with edited version<br/>
`oc replace -f /tmp/oauth.yaml`

Watch the pods being replaced for the new config to take into affect<br/>
`oc get po -w -n openshift-authentication`

Lets delete user2 from htpass file<br/>
`htpasswd -D /tmp/htpass user2`

Update the secret with new htpass<br/>
`oc set data secret/htpass-secret --from-file htpasswd=/tmp/htpass -n openshift-config`

Delete the user and identity from the system

`oc delete user user2` <br/>
`oc delete identity localusers:user2`

delete all users and identities defined in OAuth

`oc delete user --all` <br/>
`oc delete identity --all`

# 5. Users, Groups, and Authentication #

Assign cluster admin role to user<br/>
`oc adm policy add-cluster-role-to-user cluster-admin user-name`

Assign role at project/namespace level<br/>
`oc policy add-role-to-user role-name user-name -n project-name  `

Get current clusterrolebindings configured for self-provisioners<br/>
`oc get clusterrolebindings -o wide |grep -E "NAME|self-provisioners"`

Describe the clusterrolebindings and clusterrole<br/>

`oc describe clusterrolebindings self-provisioners` <br/>
`oc describe clusterrole self-provisioner` <br/>

> <span style="color:red">**clusterrolebindings = self-provisioners**</span><br/>
> **clusterrole = self-provisioner**

Remove self-provisioner role from system such that authenticated users can't create projects<br/>

`oc adm policy remove-cluster-role-from-group self-provisioner \`<br/>
&nbsp;&nbsp;`system:authenticated:oauth`

Restore self-provisioners back to cluster as original<br/>

`oc adm policy add-cluster-role-to-group \` <br/>
&nbsp;&nbsp;`--rolebinding-name self-provisioners self-provisioner system:authenticated:oauth`

Creating new groups

`oc adm group new dev-users dev1 dev2` <br/>
`oc adm group new qa-users qa1 qa2`

Assign roles at namespace/project level. You will need admin role to assign users.

`oc policy add-role-to-group edit dev-users -n namespace` <br/>
`oc policy add-role-to-group view qa-users -n namespace` <br/>
`oc policy add-role-to-user admin user1 -n namespace`

Get all the rolebindings for the current namespace<br/>
`oc get rolebindings -o wide`

# 6. Remove kubeadmin from the system #

Make sure you have assigned cluster-admin to someone else before doing this!<br/>
`oc adm policy add-cluster-role-to-user cluster-admin user-name`

Instead of removing/deleting the user, we will remove the password from the system. <br/>
`oc delete secret kubeadmin -n kube-system`

# 7. Secrets and ConfigMaps #

Create secrets from literals and apply to a deployment

`oc create secret generic secretname --from-literal key1=value1 \` <br/>
&nbsp;&nbsp;`--from-literal key2=value2` <br/>
`oc set env deployment/hello --from secret/secretname`

Mount the secret file into the pod filesystem<br/>

`oc set volume deployment/demo --add --type secret --secret-name demo-secret \` <br/>
 &nbsp;&nbsp;`--mount-path /app-secrets`

Example of setting env variables that contain sensitive data

`oc create secret generic mysql \` <br/>
 &nbsp;&nbsp;`--from-literal user=dba \` <br/>
 &nbsp;&nbsp;`--from-literal password=redhat123 \` <br/>
 &nbsp;&nbsp;`--from-literal database=test \` <br/>
 &nbsp;&nbsp;`--from-literal hostname=mysql`

`oc new-app --name mysql \` <br/>
&nbsp;&nbsp;`--docker-image registry.access.redhat.com/rhscl/mysql-57-rhel7:5.7-47`

`oc set env deployment/mysql --prefix MYSQL_ --from secret/mysql`

To use a private image in quay.io using secrets stored in files

`podman login -u quay-username quay.io`

`oc create secret generic quayio \` <br/>
&nbsp;&nbsp;`--from-file .dockerconfigjson=${XDG_RUNTIME_DIR}/containers/auth.json \` <br/>
&nbsp;&nbsp;`--type kubernetes.io/dockerconfigjson`
        
`oc secrets link default quayio --for pull`

`oc import-image php --from quay.io/quay-username/php-70-rhel7 --confirm`

# 8. Secure Routes #

Using openssl generate a private key and a public key

`openssl req -x509 -newkey rsa:2048 -nodes -keyout cert.key -out cert.crt \` <br/>
&nbsp;&nbsp;`-subj "/C=US/ST=FL/L=Tampa/O=IBM/CN=*.apps.acme.com" -days 365`

Using the key and cert create a TLS secret<br/>
`oc create secret tls demo-certs --cert cert.crt --key cert.key`

Mount the tls certs into the pod (using deployment)<br/>

`oc set volume deployment/demo --add --type=secret --secret-name demo-tls \` <br/>
&nbsp;&nbsp;`--mount-path /usr/local/etc/ssl/certs --name tls-mount`

Now create a passthrough route<br/>

`oc create route passthrough demo-https --service demo-https --port 8443 \` <br/>
&nbsp;&nbsp;`--hostname demo-https.apps.ocp4.example.com`

Using edge route with same certs<br/>

`oc expose route edge demo-https --service api-frontend --hostname api.apps.acme.com \`
 &nbsp;&nbsp;`--key cert.key --cert cert.crt`

Export the router cert in case we need to use it as a ca-cert<br/>
`oc extract secrets/router-ca --keys tls.crt -n openshift-ingress-operator --to /tmp/`

# 9. Security Contexts (SCC) and ServiceAccounts #

Get the current SCC roles defined<br/>
`oc get scc`

Get details of scc anyuid<br/>
`oc describe scc anyuid`

Create a service account in the current project and assign the anyuid priviledges to the service account.

`oc create serviceaccount svc-name` <br/>
`oc adm policy add-scc-to-user anyuid -z svc-name -n namespace` <br/>
`oc set serviceaccount deployment/demo svc-name` <br/>

review the scc priviledges needed for a pod<br/>
`oc get po/podname-756ff-9cjbj -o yaml | oc adm policy scc-subject-review -f -`

Example of gitlab being run as anyuid using serviceaccount
```
oc new-app --name gitlab --docker-image quay.io/redhattraiing/gitlab-ce:8.4.3-ce.0
oc get po/gitlab-6c5b5c5d55-gzkct -o yaml | oc adm policy scc-subject-review -f -
oc create sa gitlab-sa
oc adm policy add-scc-to-user anyuid -z gitlab-sa
oc set sa deployment/gitlab gitlab-sa
```

# 10. Limits, Quotas, and LimitRanges #

Check the node resources<br/>
`oc describe node node1`

Set resources on a deployment. Request limits are how much each request is allowed, and limit is the max allowed <br/>

`oc set resources deployment hello-world-nginx \` <br/>
&nbsp;&nbsp;`--requests cpu=10m,memory=20Mi --limits cpu=180m,memory=100Mi`

Quota is project level resources available<br/>
`oc create quota dev-quota --hard services=10,cpu=1300,memory=1.5Gi`

Cluster Quota is resources available across multiple projects<br/>

`oc create clusterquota env-qa \` <br/>
&nbsp;&nbsp;`--project-annotation-selector.openshift.io/requester=qa \` <br/>
&nbsp;&nbsp;`--hard pods=12,secrets=20,services=5`

Show all project annotations and labels<br/>
`oc describe namespace demo`

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
      default:			# default if not specified in the Pod spec
        cpu: "300m"
        memory: "200Mi"
      defaultRequest:	        # default request if not specified in the Pod spec
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

`oc scale --replicas 3 deployment/demo`

`oc autoscale dc/demo --min 1 --max 10 --cpu-percent 80`

`oc get hpa`

# 12. Readiness and Liveness #

- Readiness: How long before container is ready to serve requests. If probe fails Openshift removes the IP from the services endpoints, such that no traffic is forwarded to the container.
- Liveness: Probe to test if container is in healthy state? If probe fails, Openshift kills the conatiner and redeploys.

Probes have the following configuration settings:
- __initialDelaySeconds__: required, default value=0, how long to wait to start the probe after container starts
- __timeoutSeconds__: required, default value=1, how long to wait for the probe to finish
- __periodSeconds__: not required, default value=1, how often to probe
- __successThreshold__: not required, default value=1, min consecutive successes after failure to consider success
- __failureThreshold__: not required, default value=3, min consecutive failures to be considered a failure

Examples of setting probes:

`oc set probe dc/demo --readiness --initial-delay-seconds 20`

`oc set probe dc/webapp --readiness --get-url=http://:8080/healthz \` <br/>
&nbsp;&nbsp;`--period-seconds 10 --timeout-seconds 1 --initial-delay-seconds 30`

`oc set probe dc/mq --liveness --open-tcp 1414 --period-seconds 3 \` <br/>
&nbsp;&nbsp;`--timeout-seconds 2 --failure-threshold 3 --initial-delay-seconds 30`

`oc set probe dc/ace --liveness --get-url http://:7600/healthz \` <br/>
&nbsp;&nbsp;`--initial-delay-seconds 30 --period-seconds 10 --dry-run=client -o json | \` <br/>
&nbsp;&nbsp;&nbsp;&nbsp;`jq .spec.template.spec.containers[].livenessProbe`
```
{
  "httpGet": {
    "path": "/healthz",
    "port": 7600,
    "scheme": "HTTP"
  },
  "initialDelaySeconds": 30,
  "periodSeconds": 10
}
```
`oc set probe dc/mq --liveness --open-tcp 1414 --period-seconds 3 \` <br/>
&nbsp;&nbsp;`--timeout-seconds 2 --failure-threshold 3 --initial-delay-seconds 30 \` <br/>
&nbsp;&nbsp;&nbsp;&nbsp;`--dry-run=client -o json | jq .spec.template.spec.containers[].livenessProbe`
```
{
  "tcpSocket": {
    "port": 1414
  },
  "initialDelaySeconds": 30,
  "timeoutSeconds": 2,
  "periodSeconds": 3,
  "failureThreshold": 3
}
```

Removing probes:<br/>
`oc set probe dc/demo --remove --readiness --liveness`

# 13. Image Registry #

Built in registry is housed in openshift-image-registry namespace<br/>
`oc -n openshift-image-registry get svc`

cluster wide svc naming scheme is service-name.namespace.svc<br/>
`image-registry.openshift-image-registry.svc`

using stored images from openshift project

`oc get images` <br/>
`oc get is -n openshift | grep httpd`

`oc new-app --name nms --docker-image \` <br/>
&nbsp;&nbsp;`image-registry.openshift-image-registry.svc:5000/openshift/httpd:latest`

OR

`oc new-app --name nms \` <br/>
&nbsp;&nbsp;`image-registry.openshift-image-registry.svc:5000/openshift/httpd:latest`

OR

`oc new-app --name nms --image \` <br/>
&nbsp;&nbsp;`image-registry.openshift-image-registry.svc:5000/openshift/httpd:latest`
```
Create a service<br/>
`oc expose deployment/nms --port 8080 --target-port 8080`

# 14. Deployment Strategy #

Using a docker strategy using local build directory

`oc new-app --strategy docker --binary --name myapp` <br/>
`oc start-build myapp --from-dir . --follow` <br/>
`oc expose deployment myapp --target-port 8080 --port 80` <br/?
`oc expose svc myapp`

Using a nodejs builder

`oc new-app --binary --image-stream nodejs --name nodejs` <br/>
`oc start-build nodejs --from-dir . --follow` <br/>
`oc expose svc/nodejs`


# 15. General Troubleshooting #

Following are high level issues that are highlighted in the course
+ Limits or Quotas are exceeded
+ Taints on node (NoSchedule)
+ Route is misconfigured
+ Permissions missing, assign SA to deployment

Useful commands.
```
oc logs pod-name
oc adm top pods
oc adm top nodes
oc get events --field-selector type=Warning
oc debug pod
oc debug node/nodename
oc adm taint node node-name key-
```

Connect to a service using a debug image<br/>

`oc get svc/mysql -o jsonpath="{.spec.clusterIP}{'\n'}"`

`oc debug -t deployment/app --image registry.access.redhat.com/ubi8/ubi:8.0`

`curl -v telnet://<clusterIP-from-above>:<port-number>`

After fixing the deployment (yaml or the issue at hand), the deployment might have timed out by the time the issue was fixed. In order to push the deployment a new "rollout" might be needed. <br/>

`oc rollout latest dc/demo`
or
`oc rollout latest deployment/demo`

An example app was deployed where the endpoint wasn't working. After troubleshooting it was found the name of the service was defined with app tagname was mis spelled. Had to fix the typo to get the service working again. 

Another case was where instead of Route an Ingress with the wrong hostname was defined. When you delete the route, the route was re-created by openshift with a different route-name but still having the wrong hostname. To fix, we had to edit the Ingress configuration with the correct hostname. Once that was done, the correct route was generated and started working!
