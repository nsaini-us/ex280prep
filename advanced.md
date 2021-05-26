# Upgrade Cluster to a newer Channel #

Downgrading is not supported! So be very careful if you decide to do this!

Test your changes first
`oc patch clusterversion/version --type merge --patch '{"spec":{"channel":"stable-4.7"}}' --dry-run=client -o json | jq .spec.channel`

```
oc patch clusterversion version --type="merge" -p '{"spec":{"channel":"stable-4.6"}}'

oc adm upgrade

oc adm upgrade --to-latest=true
```
