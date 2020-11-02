
# Create cluster with AWS credentials

```
>> cat $(pwd)/k3d-aws-registries.yaml
mirrors:
  "xxxxxxxx.dkr.ecr.us-west-2.amazonaws.com":
    endpoint:
      - https://xxxxxxxx.dkr.ecr.us-west-2.amazonaws.com

configs:
  xxxxxxxx.dkr.ecr.us-west-2.amazonaws.com:
    auth:
      username: AWS
      password: xxxxxxxxx

>> k3d cluster create preet-cluster \
  --volume $(pwd)/k3d-aws-registries.yaml:/etc/rancher/k3s/registries.yaml \
  --servers 1 \
  --agents 2
```

# Verify cluster is ok

>> k3d cluster ls

