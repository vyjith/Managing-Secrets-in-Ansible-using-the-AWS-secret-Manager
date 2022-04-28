

# Managing Secrets in Ansible using the AWS secret Manager.


In this section I am going to encrypt only my variable file, you can also encrypt your main playbook.


# Description
-------------------------------------------------- 

We will be using ansible vault to encrypt the sensitive field of the application properites and store the ansible vault passowrd in AWS secret manager. We be achiving the following:-


* Ansible encrypt the secret varribale file

* While building the property file contact AWS secrets manager for vault key.

* Retrive the key from AWS secret manager.

* Build the property file and place the file (with decrypted value) in the target host.


## Prerequisites
-------------------------------------------------- 

* AWS cli installed on your server.

* jq installed on your server



## Lets encrypt the varribale file using ansible vault

-------------------------------------------------- 

```
ansible-vault encrypt credentials.yml
```


## Create a script for passing vault password
-------------------------------------------------- 

Following script will fetch the password from AWS secrets manager and pass to Ansible vault for decrypting the password used in property file


1. To make sure we communicate to AWS secrets manager we should be using IAM roles and not the AWS access key and secrets for better secuirty

2. Attact the I am role to the ec2 instance

[Create a IAM Role and policy and attach the instances to have access for aws secrets manager](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html)


## Setup AWS Secret Manager
-------------------------------------------------- 

Creating file as ansiblesecret.json with following content

```
{
    "ansible_vault_passwd": "jYJrbkjtYNqzvHBG8vLuGDtWjWB2ITJyaUL5zt1dFun3DZLzH19P5KBWpR2W5RmyiUPCGBu1zWEVVq6P"
}
```
Run the following [AWS cli command of secret manager ](https://docs.aws.amazon.com/cli/latest/reference/secretsmanager/create-secret.html) after we have assigned the IAM roles to instance. 

```

aws secretsmanager create-secret --name ansible/vaultpassword  --description "Secret" --secret-string file://ansiblesecret.json --region ap-south-1

```
> Output
```

{
    "VersionId": "d04f008c-48f1-4398-8555-3e8f94b71d0a", 
    "Name": "ansible/vaultpassword", 
    "ARN": "arn:aws:secretsmanager:ap-south-1:370200737204:secret:ansible/vaultpassword-38cA1C"
}

```

Alternatively, you can create this manually through the [AWS secret manager console](https://ap-south-1.console.aws.amazon.com/secretsmanager/home) 


## Create a retrive-secrets.sh file with the following content

```sh
#! /bin/bash

credentials=`aws secretsmanager get-secret-value --secret-id ansible/vaultpassword --region ap-south-1 | jq -r '.SecretString' | jq -r '.ansible_vault_passwd'`
echo ${credentials}

```

Providing an execution permission for the script

```
chmod +x retrive-secrets.sh
```

Lets run the playbook 

```
ansible-playbook main.yml --vault-password-file ./retrive-secrets.sh

```
> output
```
[ec2-user@ip-172-31-44-255 usercreation]$ ansible-playbook -i hosts main.yml --vault-password-file ./retrive-secrets.sh


PLAY [Creating a user] **********************************************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************************************************
ok: [localhost]

TASK [Create a login user] ******************************************************************************************************************************
[WARNING]: The input password appears not to have been hashed. The 'password' argument must be encrypted for this module to work properly.
changed: [localhost]

PLAY RECAP **********************************************************************************************************************************************
localhost                  : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```
If you are not mentioned the password file, you will get an error since your already encrypted your credentials varribale file.



