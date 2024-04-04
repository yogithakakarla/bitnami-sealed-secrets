how to deploy the Sealed Secrets Controller using Helm. The chart of interest is called sealed-secrets and it's provided by the bitnami-labs repository.

First, clone the Starter Kit Git repository, and change directory to your local copy:

```
git clone https://github.com/digitalocean/Kubernetes-Starter-Kit-Developers.git
```

```
cd Kubernetes-Starter-Kit-Developers
```

Then, add the sealed secrets bitnami-labs repository for Helm:

```
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
```

Next, update the sealed-secrets chart repository:

```
helm repo update sealed-secrets
```

Next, search the sealed-secrets repository for available charts to install:

```
helm search repo sealed-secrets
```

The output looks similar to:

NAME                            CHART VERSION   APP VERSION     DESCRIPTION
sealed-secrets/sealed-secrets   2.4.0           v0.18.1         Helm chart for the sealed-secrets controller.
Now, open and inspect the 06-kubernetes-secrets/assets/manifests/sealed-secrets-values-v2.4.0.yaml file provided in the Starter kit repository, using an editor of your choice (preferably with YAML lint support). You can use VS Code, for example:


Next, install the sealed-secrets/sealed-secrets chart, using Helm (notice that a dedicated sealed-secrets namespace is created as well):

HELM_CHART_VERSION="2.4.0"

helm install sealed-secrets-controller sealed-secrets/sealed-secrets --version "${HELM_CHART_VERSION}" \
  --namespace sealed-secrets \
  --create-namespace \
  -f "06-kubernetes-secrets/assets/manifests/sealed-secrets-values-v${HELM_CHART_VERSION}.yaml"

Notes:

A specific version for the Helm chart is used. In this case 2.4.0 is picked, which maps to the 0.18.1 version of the application. It’s good practice in general, to lock on a specific version. This helps to have predictable results, and allows versioning control via Git.
You will want to restrict access to the sealed-secrets namespace for other users that have access to your DOKS cluster, to prevent unauthorized access to the private key (e.g. use RBAC policies).
Next, list the deployment status for Sealed Secrets controller (the STATUS column value should be deployed):

helm ls -n sealed-secrets
The output looks similar to:

NAME                            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                  APP VERSION
sealed-secrets-controller       sealed-secrets  1               2021-10-04 18:25:03.594564 +0300 EEST   deployed        sealed-secrets-2.4.0   v0.18.1
Finally, inspect the Kubernetes resources created by the Sealed Secrets Helm deployment:

kubectl get all -n sealed-secrets
The output looks similar to (notice the status of the sealed-secrets-controller pod and service - must be UP and Running):

NAME                                             READY   STATUS    RESTARTS   AGE
pod/sealed-secrets-controller-7b649d967c-mrpqq   1/1     Running   0          2m19s

NAME                                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/sealed-secrets-controller   ClusterIP   10.245.105.164   <none>        8080/TCP   2m20s

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/sealed-secrets-controller   1/1     1            1           2m20s

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/sealed-secrets-controller-7b649d967c   1         1         1       2m20s

In the next step you will learn how to seal your secrets. Only your DOKS cluster can decrypt the sealed secrets, because it's the only one having the private key.

how to encrypt your generic Kubernetes secret, using kubeseal CLI. Then, you will deploy it to your DOKS cluster and see how the Sealed Secrets controller decrypts it for your applications to use.

Suppose that you need to seal a generic secret for your application, saved in the following file: your-app-secret.yaml. Notice the your-data field which is base64 encoded (it's vulnerable to attacks, because it can be very easily decoded using free tools):

apiVersion: v1
data:
  your-data: ZXh0cmFFbnZWYXJzOgogICAgRElHSVRBTE9DRUFOX1RPS0VOOg== # base64 encoded application data
kind: Secret
metadata:
  name: your-app
  
First, you need to fetch the public key from the Sealed Secrets Controller (performed only once per cluster, and on each fresh install):

```
kubeseal --fetch-cert --controller-namespace=sealed-secrets > pub-sealed-secrets.pem
```

Notes:

If you deploy the Sealed Secrets controller to another namespace (defaults to kube-system), you need to specify to the kubeseal CLI the namespace, via the --controller-namespace flag.
The public key can be safely stored in a Git repository for example, or even given to the world. The encryption mechanism used by the Sealed Secrets controller cannot be reversed without the private key (stored in your DOKS cluster only).
Next, create a sealed file from the Kubernetes secret, using the pub-sealed-secrets.pem key:

kubeseal --format=yaml \
  --cert=pub-sealed-secrets.pem \
  --secret-file your-app-secret.yaml \
  --sealed-secret-file your-app-sealed.yaml
The file content looks similar to (notice the your-data field which is encrypted now, using a Bitnami SealedSecret object):

```
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: your-app
  namespace: default
spec:
  encryptedData:
    your-data: AgCFNTLd+KD2IGZo3YWbRgPsK1dEhxT3NwSCU2Inl8A6phhTwMxKSu82fu0LGf/AoYCB35xrdPl0sCwwB4HSXRZMl2WbL6HrA0DQNB1ov8DnnAVM+6TZFCKePkf9yqVIekr4VojhPYAvkXq8TEAxYslQ0ppNg6AlduUZbcfZgSDkMUBfaczjwb69BV8kBf5YXMRmfGtL3mh5CZA6AAK0Q9cFwT/gWEZQU7M1BOoMXUJrHG9p6hboqzyEIWg535j+14tNy1srAx6oaQeEKOW9fr7C6IZr8VOe2wRtHFWZGjCL3ulzFeNu5GG0FmFm/bdB7rFYUnUIrb2RShi1xvyNpaNDF+1BDuZgpyDPVO8crCc+r2ozDnkTo/sJhNdLDuYgIzoQU7g1yP4U6gYDTE+1zUK/b1Q+X2eTFwHQoli/IRSv5eP/EAVTU60QJklwza8qfHE9UjpsxgcrZnaxdXZz90NahoGPtdJkweoPd0/CIoaugx4QxbxaZ67nBgsVYAnikqc9pVs9VmX/Si24aA6oZbtmGzkc4b80yi+9ln7x/7/B0XmyLNLS2Sz0lnqVUN8sfvjmehpEBDjdErekSlQJ4xWEQQ9agdxz7WCSCgPJVnwA6B3GsnL5dleMObk7eGUj9DNMv4ETrvx/ZaS4bpjwS2TL9S5n9a6vx6my3VC3tLA5QAW+GBIfRD7/CwyGZnTJHtW5f6jlDWYS62LbFJKfI9hb8foR/XLvBhgxuiwfj7SjjAzpyAgq
  template:
    data: null
    metadata:
      creationTimestamp: null
      name: your-app
      namespace: default
```

Note:

If you don't specify a namespace, the default one is assumed (use kubeseal --namespace flag, to change targeted namespace). 

Next, you can delete the Kubernetes secret file, because it's not needed anymore:

```
rm -f your-app-secret.yaml
```

Finally, deploy the sealed secret to your cluster:

```
kubectl apply -f your-app-sealed.yaml
```

Check that the Sealed Secrets Controller decrypted your Kubernetes secret in the default namespace:

```
kubectl get secrets
```

The output looks similar to:

NAME                  TYPE                                  DATA   AGE
your-app              Opaque                                1      31s
