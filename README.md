# Using the AWS Labs Mountpoint for Amazon S3 CSI Driver with Backblaze B2 Cloud Object Storage

The [Mountpoint for Amazon S3 CSI Driver](https://github.com/awslabs/mountpoint-s3-csi-driver) allows your Kubernetes applications to access object stores through a file system interface. Although created for use with Amazon S3, it works with S3-compatible object stores, including Backblaze B2 Cloud Object Storage.

This guide walks you through a sample deployment with two simple workloads that read and write data to a Backblaze B2 Bucket via the Mountpoint CSI driver. The [Mountpoint for Amazon S3 CSI Driver](https://github.com/awslabs/mountpoint-s3-csi-driver) repository contains full details of the driver. In particular, see [Mountpoint file system behavior](https://github.com/awslabs/mountpoint-s3/blob/main/doc/SEMANTICS.md) for a detailed description of Mountpoint's behavior and POSIX support and how they could affect your application.

Note: we have conducted basic testing to verify that the Mountpoint CSI driver works with Backblaze B2. However, [as stated in the Mountpoint for Amazon S3 documentation](https://github.com/awslabs/mountpoint-s3#compatibility-with-other-storage-services):

> Mountpoint for Amazon S3 is designed for high-performance access to the Amazon S3 service. While it may be functional against other storage services that use S3-like APIs, we aren't able to provide support for those use cases, and they may inadvertently break when we make changes to better support Amazon S3.

If you intend to use the Mountpoint CSI driver with Backblaze B2, you should verify it with your specific use case. If you encounter any incompatibilities, please open an issue on this repository.

## Prerequisites

You will need:

* Kubernetes Version >= 1.25
    * We used [Minikube](https://minikube.sigs.k8s.io/) v1.37.0 on macOS in creating this guide. The Mountpoint for S3 CSI Driver is compatible with Kubernetes versions v1.25+ and implements the CSI Specification v1.9.0. The driver supports [a range of Linux distributions](https://github.com/awslabs/mountpoint-s3-csi-driver/blob/main/README.md#distros-support-matrix) with x86-64 and arm64 architectures.
* Helm
    * We used Helm v3.19.0 in creating this guide, although the [Mountpoint CSI driver installation guide](https://github.com/awslabs/mountpoint-s3-csi-driver/blob/main/docs/INSTALL.md) also includes instructions for using Kustomize.
* [A Backblaze account with B2 enabled](https://www.backblaze.com/docs/cloud-storage-enable-backblaze-b2)
* [A Backblaze B2 Bucket](https://www.backblaze.com/docs/cloud-storage-create-and-manage-buckets)
* [A Backblaze B2 application key with access to your bucket](https://www.backblaze.com/docs/cloud-storage-create-and-manage-app-keys#create-an-app-key)
* Either of the following command-line tools:
    * [The B2 Command-Line Tool](https://www.backblaze.com/docs/cloud-storage-command-line-tools)
    * [The AWS Command-Line Tool](https://aws.amazon.com/cli/), [configured to access Backblaze B2](https://www.backblaze.com/docs/cloud-storage-use-the-aws-cli-with-backblaze-b2).

## Driver Configuration

You must create a Kubernetes secret containing your Backblaze B2 application key and its ID before you install the CSI driver.

This command creates the secret, reading the application key ID from the `AWS_ACCESS_KEY_ID` environment variable, and the application key from `AWS_SECRET_ACCESS_KEY`:

```
kubectl create secret generic aws-secret \
    --namespace kube-system \
    --from-literal "key_id=${AWS_ACCESS_KEY_ID}" \
    --from-literal "access_key=${AWS_SECRET_ACCESS_KEY}"
```

You can verify that the secret was created correctly with this command:

```
kubectl describe secret --namespace kube-system aws-secret
```

You should see output like this:

```
Name:         aws-secret
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
access_key:  31 bytes
key_id:      25 bytes
```

Note the lengths of `access_key` and `key_id`. If they are zero, or the values are transposed, then you should verify that the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` environment variables are set correctly.

## Driver Installation

Add the aws-mountpoint-s3-csi-driver Helm repository:

```
helm repo add aws-mountpoint-s3-csi-driver https://awslabs.github.io/mountpoint-s3-csi-driver
helm repo update
```

Install the CSI Driver:

```
helm upgrade --install aws-mountpoint-s3-csi-driver \
    --namespace kube-system \
    aws-mountpoint-s3-csi-driver/aws-mountpoint-s3-csi-driver
```

Verify that the CSI Driver pods are running:

```
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-mountpoint-s3-csi-driver
```

You should see output similar to this:

```
NAME                                 READY   STATUS    RESTARTS   AGE
s3-csi-controller-5867448dd5-8h4lh   1/1     Running   0          7s
s3-csi-node-sk2tt                    3/3     Running   0          7s
```

## Mount a Backblaze B2 Bucket using a Persistent Volume with a Persistent Volume Claim

Use [pv_pvc.yaml](pv_pvc.yaml) as a starting point. You must set the following fields to match your environment:

* `endpoint-url` - your bucket's endpoint, as a URL with an https prefix, for example, https://s3.us-west-004.backblazeb2.com
* `region` - the region segment of your bucket's endpoint, for example, `us-west-004`
* `prefix` - you can set a prefix to restrict access within your bucket, or remove this field to allow access to the entire bucket
* `bucketName` - your bucket's name. This guide uses `your-bucket-name` as a placeholder; replace this with your bucket's name in the example commands below.

See the [Mountpoint CSI driver configuration guide](http://github.com/awslabs/mountpoint-s3-csi-driver/blob/main/docs/CONFIGURATION.md) for full details.

Deploy the PV and PVC:

```
kubectl apply -f pv_pvc.yaml
```

### Test Writing to Backblaze B2 from a Container

Optionally, you can run a test app to write some data to Backblaze B2. Deploy [test_app.yaml](test_app.yaml):

```
kubectl apply -f test_app.yaml
```

Check that the pod is running:

```
kubectl get pod test-app
```

If the pod has not reached `Running` status, then wait a few seconds and repeat the above command.

Now look for the test file. You can use the B2 CLI:

```
b2 ls b2://your-bucket-name/some_prefix/

b2 file cat --no-progress b2://your-bucket-name/some_prefix/hello.txt
```

Alternatively, you can use the AWS CLI. This example assumes you have configured B2 as your default AWS configuration:

```
aws s3 ls s3://your-bucket-name/some_prefix/

aws s3 cp s3://your-bucket-name/some_prefix/hello.txt -
```

Now, you can stop the test app. Our simple test app does not handle the termination signal, so, by default, Kubernetes will give it a 30 second grace period to release resources. Since the test app does not need to perform any shutdown processing, you can use `--now` to shut it down immediately.

```
kubectl delete --now -f test_app.yaml
```

### Test Reading from Backblaze B2 in a Container

We'll use nginx to serve an HTML file.

First, write a minimal index file to your Backblaze B2 bucket:

```
echo '<!DOCTYPE html>
<html lang="en">
<head>
    <title>Hello from Backblaze B2!</title>
</head>
<body>
    <h1>This content is stored in Backblaze B2.</h1>
</body>
</html>' | b2 file upload --no-progress your-bucket-name - some_prefix/index.html
```

Now, deploy [nginx.yaml](nginx.yaml). The nginx deployment will use the persistent volume claim as the location for its HTML files.

```
kubectl apply -f nginx.yaml
```

Check that nginx is running:

```
kubectl get pods -l app=nginx
```

Expose the nginx deployment as a new service (defaults to port 80):

```
kubectl expose deployment nginx-deployment --type=NodePort
```

If you are using minikube, you can start a tunnel and open a browser page with a single command:

```
minikube service nginx-deployment
```

Alternatively, you can use kubectl to forward the port, then browse http://localhost:8080/

```
kubectl port-forward service/nginx-deployment 8080:80
```

Delete the nginx service and deployment:

```
kubectl delete service nginx-deployment
kubectl delete -f nginx.yaml
```

## Clean up

Remove the persistent volume claim and persistent volume:

```
kubectl delete -f pv_pvc.yaml
```

Uninstall the driver:

```
helm uninstall aws-mountpoint-s3-csi-driver --namespace kube-system
```

Delete the secret:

```
kubectl delete secret aws-secret --namespace kube-system
```

## Troubleshooting

The [Mountpoint CSI driver's troubleshooting guide](https://github.com/awslabs/mountpoint-s3-csi-driver/blob/main/docs/TROUBLESHOOTING.md) lists some common errors and gives advice for solving them, but if you are still stuck, feel free to open an issue in this repository.

## Next Steps

The Mountpoint CSI driver's GitHub repository contains several example manifests showing how to use features such as caching, running as a non-root user, and accessing multiple buckets from a single pod. See the [Static Provisioning Example directory](https://github.com/awslabs/mountpoint-s3-csi-driver/tree/main/examples/kubernetes/static_provisioning).

## License

This project is licensed under the Apache-2.0 License. It builds on the static provisioning example in the similarly licensed [Mountpoint for Amazon S3 CSI Driver](https://github.com/awslabs/mountpoint-s3-csi-driver) project.