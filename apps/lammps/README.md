# LAMMPS on AWS

LAMMPS provides steps and code samples to deploy and run LAMMPS on AWS using AWS ParallelCluster.

## Architecture

The architecture below shows the different infrastucture components that will be deployed in this solution using [AWS ParallelCluster](<https://aws.amazon.com/hpc/parallelcluster/>).
The diagram also contains a suggested network layout to deploy a HPC cluster without external access to compute and storage resources, except the head node.
You can use AWS CloudFormation to deploy the following [CloudFormation stack](<https://docs.aws.amazon.com/codebuild/latest/userguide/cloudformation-vpc-template.html>) in your account to setup the networking infrastructure.

**NOTE** This solution aims to be a quickstart and simple. As such, the HPC cluster in this solution will be deployed with headnode and compute resources with ssh external setup.
You can limit external access to compute instances by:

- setting up the network infrastucture described above
- set `use_public_ips = false` in the `config/lammps-x86-64.ini`
- set `compute_subnet_id = ` in the **vpc** section of `config/lammps-x86-64.ini` to a private subnet

![LAMMPS_Architecture](<docs/images/LAMMPS_Architecture.png> "LAMMPS Architecture")

## Deploying LAMMPS on AWS

### AWS Cloud9 Environment

[AWS Cloud9](<https://aws.amazon.com/cloud9/>) is a cloud-based integrated development environment (IDE) that lets you write, run, and debug your code with just a browser.

AWS Cloud9 contains a collection of tools that let you code, build, run, test, debug, and release software in the cloud using your internet browser. The IDE offers support for python, pip, AWS CLI, and provides easy access to AWS resources through Identity and Access Management (IAM) user credentials. The IDE includes a terminal with sudo privileges to the managed instance that is hosting your development environment. This makes it easy for you to quickly run commands and directly access AWS services.

#### Create an AWS Cloud9 environment:

The link below will create an AWS Cloud9 environment from which you will be able to create your cluster.

[![Launch Stack](<https://samdengler.github.io/cloudformation-launch-stack-button-svg/images/us-east-1.svg>)](<https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/template?stackName=hpcsolutions-cloud9&templateURL=https://awsome-hpc.s3.amazonaws.com/cf_hpc_solutions_cloud9.yaml>)

[![Launch Stack](<https://samdengler.github.io/cloudformation-launch-stack-button-svg/images/us-east-2.svg>)](<https://us-east-2.console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/create/template?stackName=hpcsolutions-cloud9&templateURL=https://awsome-hpc.s3.amazonaws.com/cf_hpc_solutions_cloud9.yaml>)

1. Open the [AWS Cloud9 console](<https://console.aws.amazon.com/cloud9>).
1. Select **MyCloud9Env**.
1. Choose **Open IDE**.

#### Disable AWS managed temporary credentials:

1. Once the Cloud9 environment is created.
1. Choose the **gear icon** in the top right to open the Prefences tab.
1. In the **Preferences** tab, choose **AWS SETTINGS**.
1. Turn off the **AWS managed temporary credentials**.
1. Close the **Preferences** tab.

### Prerequisites

Let start by downloading the LAMMPS repository containing the Infrastructure as Code on your **Cloud9 instance**.

On the **Cloud9 instance terminal**, run the script below to install the prerequisited software:

```bash
wget https://github.com/aws-samples/awsome-hpc/archive/refs/heads/main.tar.gz
mkdir -p AWSome-hpc
tar -xvzf main.tar.gz -C AWSome-hpc --strip-components 1
cd AWSome-hpc/apps/lammps
bash ./scripts/setup/install_prerequisites.sh
```

The script will install the following on the Cloud9 instance:

- [Python3 and pip](<https://pip.pypa.io/en/latest/installing/>).
- [Packer version 1.6.0 and above](<https://learn.hashicorp.com/tutorials/packer/getting-started-install?in=packer/getting-started>).
- [AWS CLI version 2](<https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html>).
- [Session Manager plugin](<https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html>).

### Install AWS ParallelCluster

Create your Python3 virtual environment

```bash
python3 -m venv .env
source .env/bin/activate
```

Install AWS ParallelCluster

```bash
pip3 install aws-parallelcluster==2.11.3
```

Set AWS Region
The command below will query the metadata of the AWS Cloud9 instance to determine in which region it has been created.

```bash
export AWS_REGION=`curl --silent http://169.254.169.254/latest/meta-data/placement/region`
```

### Building LAMMPS Image using Packer on AWS

The sample relies on packer to build an AWS Machine Image (AMI) containing an installation of LAMMPS.
It is compiled and optimized for Intel Xeon Scalable Processor using the following compiler and MPI combination:

- Intel oneAPI compiler and Intel oneAPI MPI 2021.3.0
- GNU 10.2.0 and Open MPI 4.1.0

The packer scripts are located in the amis folder and are organized by Operating System (OS) such as `\[OS\]-pc-lammps`.
The available OS for this solution are:

- Amazon Linux 2

The AMI name is generated as \[company\_name\]-\[OS\]-parallelcluster-\[parallel\_cluster\_version\]-\[application\_name\]-\[application\_version\]-\[isotime YYYMMDDHHmmss\].

Different variables are passed to packer to build the LAMMPS AMI. For simplicity, they have preset values and you can find the list and description of the variables in this [document](<docs/packer_ami_variables.md>).

#### Build an AMI

Navigate to the `amis` folder, choose the desired OS, build the AMI using packer.
You can accomplish this by typing:

```bash
# Set OS for LAMMPS to be amzn2
OS_LAMMPS=amzn2

# Go to the corresponding folder
cd amis/amzn2-pc-lammps

# Build the ami using packer
packer build \
    -var-file variables.json \
    -var aws_region=${AWS_REGION} \
    -var parallel_cluster_version=`pcluster version` \
    -var company_name=[COMPANY_NAME] \
    amzn2-pc-lammps.json
```

### Deploy AWS ParallelCluster with LAMMPS

Enable [Amazon Macie](<https://aws.amazon.com/macie/>)

```bash
aws macie2 enable-macie
```

Create AWS ParallelCluster configuration file

```bash
# Going back from where you started
cd ../../

. ./scripts/setup/create_parallelcluster_config.sh
```

Create Cluster

```bash
CLUSTER_NAME="lammps-cluster"
pcluster create ${CLUSTER_NAME} -c ./config/lammps-x86-64.ini --region ${AWS_REGION}
```

Connect to the cluster

```bash
pcluster ssh ${CLUSTER_NAME} -i ~/.ssh/${SSH_KEY_NAME} --region ${AWS_REGION}
```

## LAMMPS Performance

In this section, you will go through the steps to run and measure the performance [test case(s) provided by LAMMPS](<https://github.com/lammps/lammps/tree/stable/bench>) on AWS ParallelCluster.
The performance directory of the repository contains a sample of Slurm submission scripts to the performance test case(s) on different instances.

### Run

Once you are connected to the LAMMPS cluster, you should navigate to the `/fsx/performance` directory and submit the job script to Slurm.
Here are the steps:

```bash
cd /fsx/performance

# To run the Lennard-Jones test case on 10 x c5n.18xlarge instances
sbatch slurm-c5n-intel-lj.sh
```

The job should complete in a couple of minutes with the output and error files located in the `/fsx/performance` directory.

## Cleanup your cluster

To avoid unexpected charges to your account relative to the LAMMPS cluster, make sure you delete the cluster and associated resources.

Delete the cluster

```bash
pcluster delete ${CLUSTER_NAME} --region ${AWS_REGION}
```

**The steps below are optional if you plan to deploy a cluster with LAMMPS in the future.**

Delete remaining components of the LAMMPS solution

```bash
. ./scripts/cleanup/cleanup_solution_components.sh
```

If you wish to disable Amazon Macie

```bash
aws macie2 disable-macie
```

Delete the AWS Cloud9 environment:

1. Open the [AWS CloudFormation](<https://console.aws.amazon.com/cloudformation>).
1. Select **hpcsolutions-cloud9**.
1. Choose **Delete**.
1. Choose **Delete** to confirm deletion.
