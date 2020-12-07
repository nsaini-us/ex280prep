# Upgrade Cluster on a newer Channel #

Downgrading is not supported! So be very careful if you decide to do this!

```
oc patch clusterversion version --type="merge" -p '{"spec":{"channel":"stable-4.6"}}'

oc adm upgrade

oc adm upgrade --to-latest=true
```
