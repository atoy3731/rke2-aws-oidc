# RKE2/AWS IAM Support for Service Accounts

This is a walk-through of getting RKE2 working with an AWS IAM Identity Provider to allow Service Accounts to assume specific IAM roles.

NOTE: This is built with the assumption that your controlplane is not exposed externally. It is leveraging an S3 bucket. If your controlplane is exposed, you might have an easier time but there are obvious security implications to that.

## Creating a Static Hosted S3 Bucket

You need to create a publicly exposed S3 bucket to host 2 things: the OIDC configuration and JWKS public key. Neither of these are sensitive, so there should be no concern with the bucket being exposed.

From the AWS console, do the following:

1. Go to the S3 page.
2. Click `Create Bucket` and type in a unique name for your bucket (ie. `k8s-cluster-oidc-bucket-1234`)
3. In the `Buckets` list, find your new bucket and click on it to see its configuration, then click the `Permissions` tab.
4. In the `Block public access` box, click `Edit` and uncheck all boxes. Click `Save` and type `confirm`.
5. In the `Bucket Policy` section, click `Edit`. Paste the following in the box (substituting your BucketName):
  ```
  {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Sid": "PublicRead",
              "Effect": "Allow",
              "Principal": "*",
              "Action": [
                  "s3:GetObject",
                  "s3:GetObjectVersion"
              ],
              "Resource": "arn:aws-us-gov:s3:::YOUR_BUCKET_NAME_HERE/*"
          }
      ]
  }
  ```
6. Click the `Properties` tab and scroll to the bottom section labeled `Static website hosting`. Click `Edit`
7. Click `Enable` and type `index.html` in the `Index document` section. Leave the rest as default and click `Save`.
8. Once enabled, copy the bucket URL from the bottom of the `Static website hosting` section for the next steps (you'll also need to change `http` to `https`).

## Update API Server Flags

You now need to update the API Server flags on all controlplane nodes and configure it to use the S3 bucket as its Service Account Issuer and JWKS reference.

1. Gather the bucket URL you copied from the previous step.
2. Update your API server flags with the following:
  ```
  --service-account-issuer=https://YOUR_BUCKET_URL_HERE
  --service-account-jwks-uri=https://YOUR_BUCKET_URL_HERE/openid/v1/jwks
  ```
3. Once updated for all your controlplane nodes and the API pods in `kube-system` have reset, confirm the change is live by running (with the expected output):
  ```
  $ kubectl get pods -n kube-system -l component=kube-apiserver -o yaml | grep -e 'service-account-issuer' -e 'service-account-jwks-uri'
  - --service-account-issuer=https://example-bucket.s3-us-gov-east-1.amazonaws.com
  - --service-account-jwks-uri=https://example-bucket.s3-us-gov-east-1.amazonaws.com/openid/v1/jwks
  - --service-account-issuer=https://example-bucket.s3-us-gov-east-1.amazonaws.com
  - --service-account-jwks-uri=https://example-bucket.s3-us-gov-east-1.amazonaws.com/openid/v1/jwks
  - --service-account-issuer=https://example-bucket.s3-us-gov-east-1.amazonaws.com
  - --service-account-jwks-uri=https://example-bucket.s3-us-gov-east-1.amazonaws.com/openid/v1/jwks
  ```

## Add IAM Role to Publish to Public S3 Bucket

This part is admittedly clunky, but you need a job running within your cluster that will continuously sync your public S3 bucket with the configurations from your server. In order to do that, you'll need an IAM role with write access to that bucket. Use the following IAM policy as an example:

  ```
  {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Resource": [
                  "arn:aws:s3:::YOUR_BUCKET_NAME",
                  "arn:aws:s3:::YOUR_BUCKET_NAME/*"
              ],
              "Sid": "Stmt1464826210000",
              "Effect": "Allow",
              "Action": [
                  "s3:DeleteObject",
                  "s3:GetBucketLocation",
                  "s3:GetObject",
                  "s3:ListBucket",
                  "s3:PutObject"
              ]
          }
      ]
  }
  ```

This IAM role needs to be attached to the following job so it has the ability to write to the bucket. This can be done by either adding the role to the host EC2s as their IAM Profile or create an AWS User, attaching the Policy, gathering the AWS Key/Secret IDs, creating a K8s secret, and mounting those into the job (like I said, clunky).

## Creating the Sync Job Deployment

Create the following deployment, updating the bucket name and region to your configurations (NOTE: this implementation assumes you're using an Instance IAM Profile. If using a K8s secret to hold an AWS key/secret, you'll need to add that to the deployment as well):

```yaml
---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: "aws-oidc-s3-updater"
      labels:
        app: "aws-oidc-s3-updater"
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: "aws-oidc-s3-updater"
      template:
        metadata:
          labels:
            app: "aws-oidc-s3-updater"
        spec:
          containers:
            - name: "updater"
              image: amazon/aws-cli:latest
              env:
              - name: AWS_S3_BUCKET
                value: "YOUR_BUCKET_HERE"
              - name: AWS_DEFAULT_REGION
                value: us-gov-east-1
              command:
                - /bin/bash
                - -c
                - |
                  #!/bin/bash
                  set -e

                  TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
                  CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

                  echo ""
                  echo "[ S3 OIDC Updater ]"
                  echo ""

                  echo "Making working directory and gathering initial JWKS and OpenID Configuration.."
                  mkdir -p /tmp/working/openid/v1
                  mkdir -p /tmp/working/.well-known
                  curl --cacert $CACERT -H "Authorization: Bearer $TOKEN" https://kubernetes.default/openid/v1/jwks > /tmp/working/openid/v1/jwks 2>/dev/null
                  curl --cacert $CACERT -H "Authorization: Bearer $TOKEN" https://kubernetes.default/.well-known/openid-configuration > /tmp/working/.well-known/openid-configuration 2>/dev/null
                  aws s3 sync /tmp/working s3://$AWS_S3_BUCKET/

                  while true; do
                    sleep 10
                    JWKS=$(curl --cacert $CACERT -H "Authorization: Bearer $TOKEN" https://kubernetes.default/openid/v1/jwks 2>/dev/null)
                    OPENID_CONFIG=$(curl --cacert $CACERT -H "Authorization: Bearer $TOKEN" https://kubernetes.default/.well-known/openid-configuration 2>/dev/null)

                    if [ "$JWKS" != "$(cat /tmp/working/openid/v1/jwks)" ] || [ "$OPENID_CONFIG" != "$(cat /tmp/working/.well-known/openid-configuration)" ]; then
                      echo "Updated JWKS/OpenID Configuration found. Updating S3 bucket.."
                      echo "$JWKS" > /tmp/working/openid/v1/jwks
                      echo "$OPENID_CONFIG" > /tmp/working/.well-known/openid-configuration
                      aws s3 sync /tmp/working s3://$AWS_S3_BUCKET/
                    else
                      echo "No change detected."
                    fi
                  done
```

Once the deployment is created and running, check the logs of the pod to confirm it is pushing/syncing to the S3 bucket as expected. If not, check to make sure your IAM policy is successfully attaching to the deployment's pod.

## Checking Your Bucket

Assuming the above worked, navigate back to the S3 console, locate your bucket, and click into it.

There should now be 2 keys (`.well-known` and `openid`). Click through both to the bottom-level and you should see the following files:

* `.well-known/openid-configuration`
* `openid/v1/jwks`

Click on each of those, then click the `Object URL` link. This should download those files locally. Check them both to make sure they contain valid JSON.

## Create your Identity Provider

In your AWS console, do the following:

1. Go to the IAM console and click `Identity Providers` in the left. Click `Add provider` in the upper-right.
2. Select `OpenID Connect`.
3. For the Provider URL, use the FQDN of your S3 bucket (for instance: `https://example-bucket.s3-us-gov-east-1.amazonaws.com`)
4. Click the `Get Thumbprint` button. This should pull the required thumbprint from your S3 bucket using the publicly exposed files from the previous step.
5. Update the `Audience` to `rke2` (which is the default audience created by RKE2).
6. Click `Add provider`.

## Creating a Pod-Level IAM Role

Next you need to start creating the IAM roles that your pods will be using to perform their functions. 

1. Go back to the IAM console in your AWS account.
2. Click on `Roles` on the left and click `Create Role`.
3. For the `Trusted Entity Type`, select `Custom trust policy`, and paste the following:

  ```
  {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Effect": "Allow",
              "Principal": {
                  "Federated": "arn:aws-us-gov:iam::YOUR_ACCOUNT_ID:oidc-provider/example-bucket.s3-us-gov-east-1.amazonaws.com"
              },
              "Action": "sts:AssumeRoleWithWebIdentity",
              "Condition": {
                  "StringEquals": {
                      "example-bucket.s3-us-gov-east-1.amazonaws.com:sub": "system:serviceaccount:default:default"
                  }
              }
          }
      ]
  }
  ```
  NOTE: You'll need to update the following:

  * Your account ID
  * The Federated Principal (to reference your bucket/region).
  * The StringEquals key.
  * The StringEquals value (to match the namespace/serviceaccount to be used. For instance: `system:serviceaccount:namespace-here:service-account-here`)

4. Select whatever IAM policy is necessary. For testing, you can use the `ReadOnlyAccess` policy.
5. Click `Next`, give the role a name/description, and click `Create Role`

## Spin Up a Pod with the New Role

TODO
