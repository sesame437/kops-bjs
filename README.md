# Kops-BJS

This tutorial will walk you through building a Kubernetes cluster with [Kops](https://github.com/kubernetes/kops) in AWS Beijing or NinXia Region.

|        Name        |                     Support Kops Version                     | Support Kubernetes Version | Last Update  |
| :----------------: | :----------------------------------------------------------: | :------------------------: | ------------ |
| **pahud/kops-bjs** | [1.9.1](https://github.com/kubernetes/kops/releases/tag/1.9.1) |           1.9.6            | June03, 2018 |

### Agenda

**Prepare the AMI** 

**Install Kops and Kubectl client on your laptop**

**Create a proxy server with gost in AWS N. Virginia Region**

**Create a proxy forwarder in AWS Beijing Region**

**Create the cluster with Kops**



### Prepare the AMI 

Check the latest AMI ID from [Kops Images](https://github.com/kubernetes/kops/blob/master/docs/images.md) document and find the AMI ID in the global regions(e.g. N. Virginia).

However, as the China Beijing region already has latest CoreOS AMI, you can just check [CoreOS official EC2 AMI page](https://coreos.com/os/docs/latest/booting-on-ec2.html) and select the AMI for `cn-north-1` region, make sure you select the `HVM` AMI type. For example, current AMI ID is **ami-555a8438** ([Container Linux 1745.5.0](https://coreos.com/os/docs/1745.5.0/index.html)). Please note the latest AMI ID may change over time.

|         Region         |                          CoreOS AMI                          |
| :--------------------: | :----------------------------------------------------------: |
|  Beijing(cn-north-1)   | [ami-555a8438](https://console.amazonaws.cn/ec2/home?region=cn-north-1#launchAmi=ami-555a8438) |
| NinXia(cn-northwest-1) | [ami-06a0b464](https://console.amazonaws.cn/ec2/home?region=cn-northwest-1#launchAmi=ami-06a0b464) |



### Install Kops and Kubectl client on your laptop

- [install kops](https://github.com/kubernetes/kops/blob/master/docs/aws.md#install-kops)
- [install kubectl](https://github.com/kubernetes/kops/blob/master/docs/aws.md#install-kubectl)

### Create a proxy server with gost and AWS Fargate

click the button to create a proxy server with [gost](https://github.com/ginuerzh/gost) and AWS Fargate  in any of the following regions.

|           Region            |                     Launch Stack in VPC                      |   Runtime   |
| :-------------------------: | :----------------------------------------------------------: | :---------: |
|   **Oregon** (us-west-2)    | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/new?stackName=gost-service&templateURL=https://s3-us-west-2.amazonaws.com/pahud-cfn-us-west-2/kops-bjs/cloudformation/ecs-fargate-gost-tls-ss.yaml) | Fargate+ECS |
| **N. Virginia** (us-east-1) | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=gost-service&templateURL=https://s3-us-west-2.amazonaws.com/pahud-cfn-us-west-2/kops-bjs/cloudformation/ecs-fargate-gost-tls-ss.yaml) | Fargate+ECS |



### Create a proxy forwarder in AWS Beijing or Ninxia Region

Depending on which region you woud like to provision your Kops cluster, click the button below to create an internal **http_proxy forwarder** for your Kops cluster. This template will create one t2.micro EC2 behind ELB in your existing VPC as the proxy forwarder.



|           Region            |                     Launch Stack in VPC                      | Runtime |
| :-------------------------: | :----------------------------------------------------------: | :-----: |
|  **Beijing** (cn-north-1)   | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.amazonaws.cn/cloudformation/home?region=cn-north-1#/stacks/new?stackName=kops-proxy&templateURL=https://s3.cn-north-1.amazonaws.com.cn/kops-bjs/cloudformation/bjs.yml) | EC2+ELB |
| **NinXia** (cn-northwest-1) | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.amazonaws.cn/cloudformation/home?region=cn-northwest-1#/stacks/new?stackName=kops-proxy&templateURL=https://s3.cn-north-1.amazonaws.com.cn/kops-bjs/cloudformation/bjs.yml) | EC2+ELB |



### Create the cluster with Kops

update `create_cluster.sh` and modify the variables:

```bash
cluster_name='cluster.bjs.k8s.local'
ami='ami-xxxxxxxx'
vpcid='vpc-c1e040a5'  
```

**cluster_name** : specify your cluster name.  You can leave it as default.  Make sure the `cluster_name` ends with `.k8s.local` so it will create gossip-based cluster without using Route53, which is not available in China Beijing region.

**ami** : AMI ID. See **Prepare the AMI** above.

**vpcid**: Your existing VPC ID, in which you would launch your Kubernetes cluster with Kops.



update `env.config`

```bash
export AWS_PROFILE='bjs'
export AWS_DEFAULT_REGION='cn-north-1'
export AWS_REGION=${AWS_DEFAULT_REGION}
export KOPS_STATE_STORE=s3://pahud-kops-state-store
```

1. **AWS_PROFILE** - make sure the profile name points to your AWS Beijing Region configuration. Check *~/.aws/config* for details.

2. **AWS_DEFAULT_REGION** - specify *cn-north-1* for Beijing Region.

3. **KOPS_STATE_STORE** - you need specify an empty S3 bucket for Kops state store, make sure you change the value and points to your S3 bucket in Beijing Region.

   ​

execute the script to create the cluster:

```bash
$ bash create_cluster.sh 
```

edit your cluster

```bash
$ kops edit cluster cluster.bjs.k8s.local
```

paste the content below under the `spec` section for the cluster. Make sure you set correct httpProxy host.

```yaml
spec:
  docker:
    logDriver: ""
    registryMirrors:
        - https://registry.docker-cn.com
  egressProxy:
    httpProxy:
      host: <host>
      port: 8888
    excludes: amazonaws.com.cn,amazonaws.cn,aliyun.cn,aliyuncs.com,registry.docker-cn.com
```

(you should be able to see your httpProxy host and port info in the output of the cloudformation in Beijing Region)

update the cluster with `—yes`

```bash
kops update cluster --name cluster.bjs.k8s.local --yes
```



After a few minutes(typically 8-15min), you can validate the cluster like this:

```bash
$ kops validate cluster
Using cluster from kubectl context: cluster.bjs.k8s.local

Validating cluster cluster.bjs.k8s.local

INSTANCE GROUPS
NAME			ROLE	MACHINETYPE	MIN	MAX	SUBNETS
master-cn-north-1a-1	Master	m3.medium	1	1	cn-north-1a
master-cn-north-1a-2	Master	m3.medium	1	1	cn-north-1a
master-cn-north-1b-1	Master	m3.medium	1	1	cn-north-1b
nodes			Node	m3.medium	2	2	cn-north-1a,cn-north-1b

NODE STATUS
NAME						ROLE	READY
ip-172-31-37-81.cn-north-1.compute.internal	node	True
ip-172-31-39-42.cn-north-1.compute.internal	master	True
ip-172-31-51-46.cn-north-1.compute.internal	master	True
ip-172-31-68-190.cn-north-1.compute.internal master	True
ip-172-31-68-61.cn-north-1.compute.internal	node	True

Your cluster cluster.bjs.k8s.local is ready
```

Or get nodes list like this

```bash
$ kubectl get nodes
NAME                                           STATUS    ROLES     AGE       VERSION
ip-172-31-37-81.cn-north-1.compute.internal    Ready     node      15m       v1.9.3
ip-172-31-39-42.cn-north-1.compute.internal    Ready     master    17m       v1.9.3
ip-172-31-51-46.cn-north-1.compute.internal    Ready     master    16m       v1.9.3
ip-172-31-68-190.cn-north-1.compute.internal   Ready     master    16m       v1.9.3
ip-172-31-68-61.cn-north-1.compute.internal    Ready     node      15m       v1.9.3
```



### clean up

delete the cluster

```bash
$ kops delete cluster --name cluster.bjs.k8s.local --yes
```

And delete the two cloudformation stacks from `N.Virginia` and `Beijing` regions.





## Fast Bootstrapping with local mirror

The approach provided above will not leverage any local mirror of artifacts. If you are interested to leverage local artifacts mirror including the `gcr.io` docker hub mirror to accelerate the boostrapping, please check this table:

|           Region           |                            Guide                             |
| :------------------------: | :----------------------------------------------------------: |
|  **Beijing**(cn-north-1)   | [fastboot guide](https://github.com/pahud/kops-bjs/tree/master/bjs-fastboot) |
| **NinXia**(cn-northwest-1) | [fastboot guide](https://github.com/pahud/kops-bjs/tree/master/zhy-fastboot) |





## FAQ

Questions? check the [FAQ list here](https://github.com/pahud/kops-bjs/issues?utf8=%E2%9C%93&q=label%3AFAQ+).











