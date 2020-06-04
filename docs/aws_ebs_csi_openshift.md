# AWS EBS CSI Driver on OpenShift 4.4

## Table of Contents

- [AWS EBS CSI Driver on OpenShift 4.4](#aws-ebs-csi-driver-on-openshift-44)
  - [Table of Contents](#table-of-contents)
  - [Authors](#authors)
  - [Description](#description)
  - [Prerequisites](#prerequisites)
  - [Setup Credentials](#setup-credentials)
  - [Install AWS EBS CSI Driver](#install-aws-ebs-csi-driver)
  - [Create StorageClass for AWS EBS](#create-storageclass-for-aws-ebs)

## Authors

- Jared Hocutt ([@jaredhocutt](https://github.com/jaredhocutt))
- Dan Clark ([@dmc5179](https://github.com/dmc5179))

## Description

This guide describes the steps required to isntall and configure the AWS EBS
CSI driver on OpenShift 4.4.

Out of the box, the AWS EBS CSI driver does not work due to being unable to
access the AWS metadata endpoint (i.e. 169.254.169.254). To work around the
issue, the pods that get deployed need to set `hostNetwork: true` and then
workaround a couple of issues that arise by making that change.

The details of the root cuase are captured in this Bugzilla:

https://bugzilla.redhat.com/show_bug.cgi?id=1718389

## Prerequisites

1. You must be logged into the OpenShift cluster.

2. You must already have Helm 3 installed.

   See here for details: https://helm.sh/docs/intro/install/

3. You must have AWS credentials (access key and secret key) that you would
   like the AWS EBS CSI driver to use when communicating with the AWS API.

   The permissions attached to the credentials you are using should meet the
   following minimum permissions.

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "ec2:AttachVolume",
           "ec2:CreateSnapshot",
           "ec2:CreateTags",
           "ec2:CreateVolume",
           "ec2:DeleteSnapshot",
           "ec2:DeleteTags",
           "ec2:DeleteVolume",
           "ec2:DescribeAvailabilityZones",
           "ec2:DescribeInstances",
           "ec2:DescribeSnapshots",
           "ec2:DescribeTags",
           "ec2:DescribeVolumes",
           "ec2:DescribeVolumesModifications",
           "ec2:DetachVolume",
           "ec2:ModifyVolume"
         ],
         "Resource": "*"
       }
     ]
   }
   ```

## Setup Credentials

1. Create secret `aws-secret.yaml` that contains your AWS credentials by
   setting the values for `stringData.key_id` and `stringData.access_key`.

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: aws-secret
     namespace: kube-system
   stringData:
     key_id: ""
     access_key: ""
   ```

2. Apply the secret.

   ```bash
   oc apply -f aws-secret.yaml
   ```

## Install AWS EBS CSI Driver

The AWS EBS CSI driver does not work out-of-the-box on OpenShift and the
deployment requires slight modification to make it work. Therefore, the AWS EBS
CSI driver should be installed using the Helm chart as it is easier to modify
than the operator.

1. Download the AWS EBS CSI driver Helm chart.

   ```bash
   curl -LO https://github.com/kubernetes-sigs/aws-ebs-csi-driver/releases/download/v0.5.0/helm-chart.tgz
   ```

2. Unpack the Helm chart so you can make modifications.

   ```bash
   tar xvf helm-chart.tgz
   ```

   Example output:

   ```text
   aws-ebs-csi-driver/
   aws-ebs-csi-driver/values.yaml
   aws-ebs-csi-driver/templates/
   aws-ebs-csi-driver/templates/_helpers.tpl
   aws-ebs-csi-driver/templates/csidriver.yaml
   aws-ebs-csi-driver/templates/deployment.yaml
   aws-ebs-csi-driver/templates/rbac.yaml
   aws-ebs-csi-driver/templates/statefulset.yaml
   aws-ebs-csi-driver/templates/serviceaccount.yaml
   aws-ebs-csi-driver/templates/NOTES.txt
   aws-ebs-csi-driver/templates/daemonset.yaml
   aws-ebs-csi-driver/.helmignore
   aws-ebs-csi-driver/Chart.yaml
   ```

3. Edit `aws-ebs-csi-driver/templates/daemonset.yaml` and make the following
   modifications.

   - Change the `containerPort` for `healthz` of the `ebs-plugin` container on
     **line 55** from `9808` to `19808`.

     Before:

     ```yaml
     ports:
       - name: healthz
         containerPort: 9808
         protocol: TCP
     ```

     After:

     ```yaml
     ports:
       - name: healthz
         containerPort: 19808
         protocol: TCP
     ```

   - Add argument `--health-port=19808` to the `liveness-probe` container after
     the `--csi-address=/csi/csi.sock` argument on **line 88**.

     Before:

     ```yaml
     args:
       - --csi-address=/csi/csi.sock
     ```

     After:

     ```yaml
     args:
       - --csi-address=/csi/csi.sock
       - --health-port=19808
     ```

4. Edit `aws-ebs-csi-driver/templates/deployment.yaml` and make the following
   modifications.

   - Change the `containerPort` for `healthz` of the `ebs-plugin` container on
     **line 73** from `9808` to `29808`.

     Before:

     ```yaml
     ports:
       - name: healthz
         containerPort: 9808
         protocol: TCP
     ```

     After:

     ```yaml
     ports:
       - name: healthz
         containerPort: 29808
         protocol: TCP
     ```

   - Add argument `--health-port=29808` to the `liveness-probe` container after
     the `--csi-address=/csi/csi.sock` argument on **line 145**.

     Before:

     ```yaml
     args:
       - --csi-address=/csi/csi.sock
     ```

     After:

     ```yaml
     args:
       - --csi-address=/csi/csi.sock
       - --health-port=29808
     ```

   - Configure Pod to use `hostNetwork` by adding `hostNetwork: true` in the
     Pod `spec` on **line 23**.

     Before:

     ```yaml
     spec:
       nodeSelector:
     ```

     After:

     ```yaml
     spec:
       hostNetwork: true
       nodeSelector:
     ```

5. Repack the Helm chart.

   ```bash
   tar czvf helm-chart-new.tgz aws-ebs-csi-driver/
   ```

6. Deploy the modified AWS EBS CSI driver Helm chart.

   ```bash
   helm install aws-ebs-csi-driver ./helm-chart-new.tgz \
     --namespace kube-system \
     --set enableVolumeScheduling=true \
     --set enableVolumeResizing=true \
     --set enableVolumeSnapshot=true
   ```

## Create StorageClass for AWS EBS

1. Create StorageClass `ebs-sc.yaml`.

   ```yaml
   ---
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
     annotations:
       storageclass.kubernetes.io/is-default-class: "true"
     name: ebs-sc
   provisioner: ebs.csi.aws.com
   volumeBindingMode: Immediate
   ```

2. Apply the StorageClass.

   ```bash
   oc apply -f ebs-sc.yaml
   ```
