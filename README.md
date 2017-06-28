# Kube-Registry

&nbsp;

With this guide you will be able to setup Docker Private Registry inside Kubernetes, using AWS S3 as a storage endpoint. As an example, we will assume that we use AWS Kubernetes cluster maintained by KOPS.

&nbsp;

### Creating S3 Bucket

First of all, we need to create an S3 bucket located in the same region where your cluster is set up. Let's name it "*s3-storage*". We also assume you are familiar with AWS S3 interface and creating it should not be a problem.

&nbsp;

### Creating AWS User

After bucket is created, you need to create a new policy, new user, and link that policy to the user. IAM service is the instrument we will be using. Also, this user's permissions will be quite limited to S3 only.

&nbsp;

* First, we will create a policy. Go to **IAM** -> **Policies** -> **Create Policy**. Pick *AmazonS3FullAccess* template, switch to **JSON view**, then click **Edit**, and change it to the following:

&nbsp;

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetBucketLocation",
                "s3:ListBucketMultipartUploads"
            ],
            "Resource": "arn:aws:s3:::s3-storage"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:ListMultipartUploadParts",
                "s3:AbortMultipartUpload"
            ],
            "Resource": "arn:aws:s3:::s3-storage/*"
        }
    ]
}
```

&nbsp;

Name this policy "*registry-access-policy*". We're done here.

&nbsp;

* To create a user, go to **IAM** -> **Users** -> **Add user**. Choose 'programmatic access' when creating a user to generate **AccessKey** / **AccessId** we will later need. Let's name this user *registry-access*. Don't forget to write down **Access Key** / **Key ID** at the end of the process.
* When user is created, we need to attach our *registry-access-policy* to it. Pick your *registry-access* user in **IAM** interface, then click **Add Permission** -> **Attach existing policy directly**, and pick *registry-access-policy* from the list of available policies.

&nbsp;

That's where we're done with user creation process.

&nbsp;

### Look at what you will be deploying

&nbsp;

* All files needed for Kube deployment can be found here: https://github.com/kuberstack/kube-registry
Let's clone it:

&nbsp;

```
$ git clone https://github.com/kuberstack/kube-registry.git
$ cd kube-registry
```

&nbsp;

* You will see two files which describe Kubernetes objects: **service** & **replication controller**.

&nbsp;

```
svc-kube-registry.yaml
rc-kube-registry.yaml
```

&nbsp;

### Deploying (finally)

&nbsp;

* Setting up **service** can be tricky and depends on your environment. Sometimes it is enough to just create a simple service which makes service available directly, sometimes you need nginx-ingress / nginx, etc. During this guide we will use AWS ELB which does not require any additional settings (except for DNS records, which I will mark out additionally).

&nbsp;

```
type: LoadBalancer
```

&nbsp;

* To create a service, simply execute:

&nbsp;

```
$ kubectl --namespace=default create -f svc-registry.yaml
```

&nbsp;

#### *Before creating a replication controller, you need to perform a series of tasks to proceed — you have to generate an SSL certificate and also create htpasswd file which will be used for BasicAuth mechanism.*

&nbsp;

### SSL

&nbsp;

* If you don't have any ready SSL certs, you can get one from LetsEncrypt using Docker Certbot image:

&nbsp;

```
$ docker run --rm -v /var/www/html:/var/www/html -v /etc/letsencrypt/:/etc/letsencrypt/ -ti deliverous/certbot certonly -d registry.domain.com --webroot -w /var/www/html/
```

&nbsp;

Note that your 'registry.domain.com' should exist as an A-record, otherwise it won't work. Upon success, you will get **fullchain.pem** & **privkey.pem** inside /etc/letsencrypt/live/ which need to be compiled into **Kubernetes secret**:

&nbsp;

```
$ cd /etc/letsencrypt/live/registry.domain.com/; cp fullchain.pem domain.crt; cp privkey.pem domain.key; kubectl -n default create secret generic registry-ssl-certs --from-file=domain.key --from-file=domain.crt
```

&nbsp;

Then, you need to 'add' these certs to your **replication controller** as a volume, just like it described below:

&nbsp;
   
```   
	  volumes:
      - name: registry-ssl-certs
        secret:
          defaultMode: 420
          secretName: registry-ssl-certs
        volumeMounts:
        - mountPath: /certs/
          name: registry-ssl-certs
          readOnly: true
```

&nbsp;

### Setting up authorization

&nbsp;

* htpasswd file can be generated with official Docker Registry image:

&nbsp;

```
$ docker run --entrypoint htpasswd registry:2 -Bbn username password > htpasswd
```

And then you also need to wrap it up into a secret:

```
$ kubectl -n default create secret generic htpasswd-file --from-file=htpasswd
```

&nbsp;

Then, you need to 'add' this file to your **replication controller** as a volume, just like it described below:

&nbsp;

```
      volumes:
      - name: htpasswd-file
        secret:
          defaultMode: 420
          secretName: htpasswd-file
        volumeMounts:
        - mountPath: /auth/
          name: htpasswd-file
          readOnly: true
```

&nbsp;

### Starting Replication Controller (yay!)

&nbsp;

* Now we need to finally create **replication controller**, which will run **kube-registry** pod. *You really have to pay attention to EnvVars because that's how you control your Registry*. Main EnvVars we will be using:

&nbsp;

```
REGISTRY_STORAGE: s3 # enables S3 storage driver
REGISTRY_AUTH: htpasswd # describes which auth logic will be used
REGISTRY_STORAGE_S3_ACCESSKEY: AKIAXXXXXXXXX # AccessKey that we have created before
REGISTRY_STORAGE_S3_SECRETKEY:  RpQ6E/XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX # SecretKey that we have created before
REGISTRY_STORAGE_S3_REGION: us-east-1 # AWS region with your cluster and S3 bucket
REGISTRY_STORAGE_S3_BUCKET: s3-storage # S3 bucket name
REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd # htpasswd path
REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt # SSL cert path
REGISTRY_HTTP_TLS_KEY: /certs/domain.key # SSL cert key path
```

&nbsp;

* Now, double-check everything we've made before: user, policy, permissions, service, SSL, htpasswd. If everything's alright, you're good to go:

&nbsp;

```
$ kubectl -n default create -f rc-registry.yaml
```

&nbsp;

### Setting up DNS and Grand Finale

&nbsp;

* You can verify that your **replication controller** works by checking if pod is running:

&nbsp;

```
$ kubectl -n default get pods | grep registry
NAME READY STATUS RESTARTS AGE
kube-registry-c98v7 1/1 Running 0 5m
```

&nbsp;

* When **replocation controller** is up and running, you have to create an A-record so your registry will be available via FQDN like registry.domain.com. In our guide, we're using AWS Route53 as a DNS zone hoster. You can retrieve ELB URL you will need for A-record later by running:

&nbsp;

```
$ kubectl -n default describe svc kube-registry | grep LoadBalancer

Type:                   LoadBalancer
LoadBalancer Ingress:   unique-id-here.us-east-1.elb.amazonaws.com
```

&nbsp;

Take this URL and create a new record set (type = A-record) inside Route53, don't forget to check 'is an Alias = yes', and point it to unique-id-here.us-east-1.elb.amazonaws.com (ELB URL we've retrieved before).

&nbsp;

**That's where it finally ends.**

&nbsp;

* You can now test your registry and also check how your BasicAuth works:

&nbsp;

```
$ docker login registry.domain.com
Login:
Password:
Login Successful. ## That's the output in case of success
```

&nbsp;

```
$ docker tag busybox registry.domain.com/busybox
$ docker push registry.domain.com/busybox
The push refers to a repository [registry.domain.com/busybox]
4ac76077f2c7: Successfully pushed
latest: digest: sha256:af688dc06c671809afb41f8be4f2ffaa28d9454f027198f854e4d38883c0e685 size: 2136 ## That's the output you'll see on successful push.
```
