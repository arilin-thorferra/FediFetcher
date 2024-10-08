# Running FediFetcher using a Kubernetes cronjob

> [!NOTE]
> 
> The below are not step-by-step instructions. We assume that you mostly know what you are doing.

You should first create a k8s secret, in order to then expose the token as an environment variable (this also avoids anything which might log the command line from including the sensitive value):

```bash
kubectl create secret generic fedifetcher \
    --from-literal=server_domain=example.com \
    --from-literal=token="<token>"
```

Now define the cronjob, and don't forget to define your PVCs:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: fedifetcher
spec:
  schedule: "*/15 * * * *"
  failedJobsHistoryLimit: 5
  successfulJobsHistoryLimit: 5
 concurrencyPolicy: Forbid
  jobTemplate:
    spec:
        template:
            spec:
                restartPolicy: Never
                containers:
                - name: fedifetcher
                  image: ghcr.io/nanos/fedifetcher:latest
                  imagePullPolicy: IfNotPresent
                  env:
                    - name: FF_HOME_TIMELINE_LENGTH
                      value: "200"
                    - name: FF_MAX_FOLLOWERS
                      value: "10"

                    # Add any other options below as described in in the README.md file

                    # If you don't want to use a PVC you may comment the next two lines, but that will significantly 
                    # affect performance, and is NOT recommended
                    - name: FF_STATE_DIR
                      value: "/data/"
                    - name: FF_SERVER
                      valueFrom: 
                        secretKeyRef:
                            name: fedifetcher
                            key: server_domain
                            optional: false
                    - name: FF_ACCESS_TOKEN
                      valueFrom: 
                         secretKeyRef:
                            name: fedifetcher
                            key: token
                            optional: false
                # Comment the lines below if you do not use a PVC, but that will significantly 
                # affect performance and is NOT recommended
                volumeMounts:
                    - mountPath: /data
                      name: fedifetcher
                      readOnly: false 
                volumes:
                    - name: fedifetcher               
                      persistentVolumeClaim:
                       claimName: fedifetcher
```

