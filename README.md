# Ansible EC2 Server for Patching using AWS_EC2 and AWS_SSM

## About the Project
In an AWS environment, you have the option of using AWS Session Manager to access any EC2 instance without ever needing to configure SSH keys. This project demonstrates how to set up an Ansible EC2 server for patching using AWS EC2 and AWS SSM, leveraging AWS Session Manager for secure and seamless access. The best part is that this functionality is already included in the standard Ansible installation.

## Getting Started

- [Ansible EC2 Server for Patching using AWS\_EC2 and AWS\_SSM](#ansible-ec2-server-for-patching-using-aws_ec2-and-aws_ssm)
  - [About the Project](#about-the-project)
  - [Getting Started](#getting-started)
    - [1. Create S3 Bucket](#1-create-s3-bucket)
    - [2. Create IAM Role](#2-create-iam-role)
      - [2.1 Create Custom Policy for S3 Bucket Access](#21-create-custom-policy-for-s3-bucket-access)
      - [2.2 Create Ansible Role and add next Policies:](#22-create-ansible-role-and-add-next-policies)
    - [3. Create Ansible server](#3-create-ansible-server)
      - [3.1 Launch EC2 with Ubuntu 24.04](#31-launch-ec2-with-ubuntu-2404)
      - [3.2 Install Ansible and requirements](#32-install-ansible-and-requirements)
        - [\> Update Ubuntu packages:](#-update-ubuntu-packages)
        - [\> Install Boto and Boto3:](#-install-boto-and-boto3)
        - [\> Install Ansible and amazon.aws collection:](#-install-ansible-and-amazonaws-collection)
        - [\> Install AWS CLI](#-install-aws-cli)
        - [\> Install Amazon Session manager](#-install-amazon-session-manager)
    - [4 Ansible Playbook configuration:](#4-ansible-playbook-configuration)
      - [4.1 Dynamic Inventory without Hostnames](#41-dynamic-inventory-without-hostnames)
        - [\> Create hosts.aws\_ec2.yml. Name MUST be with ending ``.aws_ec2.yml ``](#-create-hostsaws_ec2yml-name-must-be-with-ending-aws_ec2yml-)
      - [4.2 Vars](#42-vars)
        - [\> Necessarily Present in vars or vars.yml](#-necessarily-present-in-vars-or-varsyml)
      - [5.1 Ansible Config (ansible.conf)](#51-ansible-config-ansibleconf)
      - [5.2 How to use AWS\_EC2](#52-how-to-use-aws_ec2)
      - [5.3 Connect servers to SSM](#53-connect-servers-to-ssm)
        - [5.3.1 Create AWSEC2RoleforSSM Role and add next Policies:](#531-create-awsec2roleforssm-role-and-add-next-policies)
        - [5.3.2 Attaching AWSEC2RoleforSSM to all hosts servers](#532-attaching-awsec2roleforssm-to-all-hosts-servers)
      - [5.4 Run Ansible playbook](#54-run-ansible-playbook)
        - [Before you run ansible playbook download from github project **patching** and go in:](#before-you-run-ansible-playbook-download-from-github-project-patching-and-go-in)
        - [Run Ansible playbook](#run-ansible-playbook)
      - [5.5 Debug:](#55-debug)
        - [You can check all errors:](#you-can-check-all-errors)
        - [You can check all servers:](#you-can-check-all-servers)




### 1. Create S3 Bucket
Create a simple bucket with **standard permissions** and name it **ansible3**.

### 2. Create IAM Role

#### 2.1 Create Custom Policy for S3 Bucket Access
Create a policy named **Ansible3**. Copy the policy below and paste it as JSON.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowSSMAccess",
            "Effect": "Allow",
            "Action": [
                "ssm:GetConnectionStatus",
                "ssm:ResumeSession",
                "ec2:DescribeInstances",
                "ssm:DescribeSessions",
                "ssm:TerminateSession",
                "ssm:StartSession"
            ],
            "Resource": "*"
        },
        {
            "Sid": "AllowAnsibleS3BucketAccess",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::ansible3/*"
        }
    ]
}
```

#### 2.2 Create Ansible Role and add next Policies:
Create a role for example named: **"AnsibleEC2S3SMM"** and attach the following policies:

* `AmazonEC2FullAccess`
* `AmazonEC2RoleforSSM`
* `AmazonS3FullAccess`
* `AmazonSSMFullAccess`
* `Ansible3` (our custom Policy which we created)


### 3. Create Ansible server

#### 3.1 Launch EC2 with Ubuntu 24.04
AMI: **ami-04b70fa74e45c3917**

AMI name: **ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-20240423**

Instance type: **t2.micro**

Region: **us-east-1**

IAM: **AnsibleEC2S3SMM**

#### 3.2 Install Ansible and requirements

##### > Update Ubuntu packages:
```bash
sudo apt update
```
##### > Install Boto and Boto3:
```bash
sudo apt install python3-pip -y
```
```bash
sudo apt install python3-boto -y
sudo apt install python3-boto3 -y
```
##### > Install Ansible and amazon.aws collection:
```bash
sudo apt install ansible -y
```
```bash
ansible-galaxy collection install amazon.aws
```
##### > Install AWS CLI
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
```
##### > Install Amazon Session manager
```bash
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" -o "session-manager-plugin.deb"
```
```bash
sudo dpkg -i session-manager-plugin.deb
```
### 4 Ansible Playbook configuration:
#### 4.1 Dynamic Inventory without Hostnames

##### > Create hosts.aws_ec2.yml. Name MUST be with ending ``.aws_ec2.yml ``
```
plugin: aws_ec2
filters:
  instance-state-name: running
  tag:Stage:
    - Development
compose:
  ansible_host: instance_id
```

#### 4.2 Vars

##### > Necessarily Present in vars or vars.yml
```
ansible_connection: aws_ssm
# ansible_aws_ssm_region: us-east-1
ansible_aws_ssm_bucket_name: ansibles3
ansible_aws_ssm_timeout: 500
```
Where:

- ***ansible3*** - is our bucket name
- ***500*** - is the timeout in seconds. Without this, the default is 60 seconds. We need to increase the time, for example, when creating a snapshot or requiring more time for updating.

#### 5.1 Ansible Config (ansible.conf)

```
[defaults]
inventory = hosts
host_key_checking = False
stdout_callback = debug

[privilege_escalation]
gather_facts = yes
become = true

[inventory]
enable_plugins = aws_ec2, aws_ssm
```
#### 5.2 How to use AWS_EC2

When you using SSM method for connecting to servers and and whant to gether aws meta informationm you need to delegate it to the ansible server. Example:

```
- name: Get EC2 instance information
  ec2_instance_info:
    region: "{{ aws_region }}"
    instance_ids:
      - "{{ instance_id }}"
  register: ec2_instance
  delegate_to: "{{ ansible_server_id }}"


- name: Create snapshots for all volumes
  ec2_snapshot:
    volume_id: "{{ item.ebs.volume_id }}"
    snapshot_tags: 
      Name: Patching
    description: "Snapshot of {{ item.ebs.volume_id }}"
    region: "{{ aws_region }}"
    last_snapshot_min_age: 60
  loop: "{{ ec2_instance.instances[0].block_device_mappings }}"
  when: item.ebs.volume_id is defined
  register: snapshots_info
  delegate_to: "{{ ansible_server_id }}"
```

#### 5.3 Connect servers to SSM
##### 5.3.1 Create AWSEC2RoleforSSM Role and add next Policies:
Create a role for example named: **"AWSEC2RoleforSSM"** and attach the following policies:

* `AmazonEC2RoleforSSM`
* `AmazonSSMManagedInstanceCore`

##### 5.3.2 Attaching AWSEC2RoleforSSM to all hosts servers 
To check if SSM has all access to all servers you need to go to:

**Systems manager** > **Fleet manager and** and here you will see all nodes connected to SSM.

#### 5.4 Run Ansible playbook

##### Before you run ansible playbook download from github project **patching** and go in:
```
cd patching
```
##### Run Ansible playbook

```
sudo ansible-playbook -i hosts.aws_ec2.yml build.yml
```
#### 5.5 Debug:
##### You can check all errors:
```
sudo ansible-playbook -i hosts.aws_ec2.yml build.yml -vvvv
```
##### You can check all servers:
```
sudo ansible-inventory -i hosts.aws_ec2.yml list
```
