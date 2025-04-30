# Crossplane S3 Bucket Changelogs Example

## Prerequisites
```
kind create cluster
helm install crossplane --namespace crossplane-system --create-namespace crossplane-stable/crossplane
```

## Install and Configure Provider
```
kubectl apply -f drc.yaml
kubectl apply -f provider.yaml
kubectl get pkg
```

## Configure AWS Credentials
```
kubectl create secret generic aws-secret -n crossplane-system --from-file=creds=./aws-credentials.txt
kubectl apply -f providerconfig.yaml
```

## Create S3 Bucket

This example creates an S3 bucket, along with the necessary configuration to
create private bucket ACLs. S3 discourages the use of ACLs nowadays, so some
[extra configuration is
needed](https://www.learnaws.org/2023/08/26/aws-s3-bucket-does-not-allow-acls/),
such as ownership controls and public access block configuration.
```
kubectl create -f bucket.yaml
kubectl apply -f bucket-ownership-controls.yaml
kubectl apply -f bucket-public-access-block.yaml
kubectl apply -f bucket-acl.yaml
```

## Examine the change logs

Check the change logs to see the objects create operations being recorded, using
a `jq` filter to look at a subset of the fields:
```
kubectl -n crossplane-system logs -l pkg.crossplane.io/provider=provider-aws-s3 --tail=500 -c changelogs-sidecar | jq '.timestamp + " " + .provider + " " + .name + " " + .operation'
```

Here's an example of what these filtered logs look like:
```
"2025-04-30T15:18:37Z provider-kubernetes:v1.21.1-18.ga1784c7e5.dirty crossplane-bucket-zzgts OPERATION_TYPE_CREATE"
"2025-04-30T15:18:50Z provider-kubernetes:v1.21.1-18.ga1784c7e5.dirty crossplane-bucket-ownership-controls OPERATION_TYPE_CREATE"
"2025-04-30T15:18:51Z provider-kubernetes:v1.21.1-18.ga1784c7e5.dirty crossplane-bucket-acl OPERATION_TYPE_CREATE"
"2025-04-30T15:18:51Z provider-kubernetes:v1.21.1-18.ga1784c7e5.dirty crossplane-bucket-public-access-block OPERATION_TYPE_CREATE"
```

Some other useful change logs examination commands:
```
# check logs for provider runtime container
kubectl -n crossplane-system logs -l pkg.crossplane.io/provider=provider-aws-s3 --tail=500

# check logs for changelogs sidecar container in raw format
kubectl -n crossplane-system logs -l pkg.crossplane.io/provider=provider-aws-s3 --tail=500 -c changelogs-sidecar

# check just the last change logs entry in nice JSON format
kubectl -n crossplane-system logs -l pkg.crossplane.io/provider=provider-aws-s3 --tail 1 -c changelogs-sidecar | jq .
```

## Clean-up

Clean up all the resources, if there are any hangs, make sure the bucket is
deleted and the rest should resolve themselves:
```
kubectl delete -f bucket-ownership-controls.yaml
kubectl delete -f bucket-public-access-block.yaml
kubectl delete bucket.s3.aws.upbound.io -l testing.crossplane.io/scenario=changelogs-bucket
kubectl delete -f bucket-acl.yaml
```

Check the change logs to see these delete operations being recorded. Depending
on if the bucket is deleted first, some of the other configuration objects may
not need an AWS API call to be deleted:
```
kubectl -n crossplane-system logs -l pkg.crossplane.io/provider=provider-aws-s3 --tail=500 -c changelogs-sidecar | jq '.timestamp + " " + .provider + " " + .name + " " + .operation'
```

Ensure no more resources are left behind:
```
kubectl get managed
```