# Upgrade Cluster on a newer Channel #

```
oc patch clusterversion version --type="merge" -p '{"spec":{"channel":"stable-4.6"}}'
```
