# Mount Amazon FSx on Amazon EC2

This sample shows how to mount Amazon FSx Lustre on Amazon EC2 during the initialization by using AWS CloudFormation.

## Deployment

First, you need to [create a KeyPair on Amazon Managed Console](https://console.aws.amazon.com/ec2/v2/home#KeyPairs).
We use the KeyPair name in the following step to connect to the instance.

Next, deploy Amazon VPC with subnets.

```bash
aws cloudformation deploy --template templates/vpc.yaml --stack-name MountAmazonFSxOnAmazonEc2Vpc
```

Next, deploy Amazon FSx Lustre.
It requires Amazon VPC id, Amazon VPC CIDR block, and one subnet id as parameters.
You need to import them before the deployment.
We choose a public subnet as the parameter where Amazon FSx Lustre will be deployed since we'll demonstrate the behavior in an instance on the subnet.

```bash
# Importing the parameter values
export VPCSTACK=MountAmazonFSxOnAmazonEc2Vpc
export VPCID=$(aws cloudformation describe-stacks --stack-name $VPCSTACK --query "Stacks[0].Outputs[?OutputKey=='VpcId'].OutputValue" --output text)
export VPCCIDRBLOCK=$(aws cloudformation describe-stacks --stack-name $VPCSTACK --query "Stacks[0].Outputs[?OutputKey=='VpcCidrBlock'].OutputValue" --output text)
export SUBNETID=$(aws cloudformation describe-stacks --stack-name $VPCSTACK --query "Stacks[0].Outputs[?OutputKey=='PublicSubnet1Id'].OutputValue" --output text)

# Execute the deployment
aws cloudformation deploy --template templates/fsx-luster.yaml --stack-name MountAmazonFSxOnAmazonEc2FSx --parameter-overrides VpcId=$VPCID VpcCidrBlock=$VPCCIDRBLOCK SubnetId=$SUBNETID
```

Finally, deploy an instance onto the public subnet.
It requires Amazon VPC id, subnet id, EC2 KeyPair name, SSH allowed network CIDR block, Amazon FSx id, and AMI of the instance.
The Amazon VPC id, subnet id, and Amazon FSx id can be imported by the stacks we deployed.
The EC2 KeyPair name is the name of the KeyPair which we created at the first step.
The SSH allowed network CIDR block is your network CIDR.
If you omit the parameter, it allows ssh from internet. (The default value is "0.0.0.0/0".)
The AMI of the Amazon Linux 2 can be found by `aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2`.

```bash
# Importing the parameter values
export VPCSTACK=MountAmazonFSxOnAmazonEc2Vpc
export VPCID=$(aws cloudformation describe-stacks --stack-name $VPCSTACK --query "Stacks[0].Outputs[?OutputKey=='VpcId'].OutputValue" --output text)
export SUBNETID=$(aws cloudformation describe-stacks --stack-name $VPCSTACK --query "Stacks[0].Outputs[?OutputKey=='PublicSubnet1Id'].OutputValue" --output text)
export FSXSTACK=MountAmazonFSxOnAmazonEc2FSx
export FSXID=$(aws cloudformation describe-stacks --stack-name $FSXSTACK --query "Stacks[0].Outputs[?OutputKey=='FsxId'].OutputValue" --output text)
export AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2 | jq .Parameter.Value -r)
export NETWORKCIDR="<REPLACE_YOUR_NETWORK_CIDR>"
export KEYNAME="<REPLACE_YOUR_KEYPAIR_NAME>"

# Execute the deployment
aws cloudformation deploy --template templates/instance.yaml --stack-name MountAmazonFSxOnAmazonEc2Instance --parameter-overrides VpcId=$VPCID SubnetId=$SUBNETID FsxId=$FSXID SshAllowedCidr=$NETWORKCIDR KeyName=$KEYNAME AMI=$AMI --capabilities CAPABILITY_IAM
```

## Demonstration

Let's try importing and exporting a file on the EC2 instance between S3 bucket via FSx Lustre.
First, you need to ssh to the instance.

```bash
ssh -i <REPLACE_PATH_TO_KEYPAIR.pem> ec2-user@<REPLACE_INSTANCE_PUBLIC_IP>
```

Check whether the FSx mount is done correctly.
See UseData of [/templates/instance.yaml](/templates/instance.yaml) to know how to mount Amazon FSx Lustre during the initialization.

```bash
df -H
```

There should be `/mnt/fsx` directory which is mounted by FSx Luster.
Check the content of the directory.
You can confirm that there're no files now.

```bash
sudo ls -la /mnt/fsx
```

### Import from S3 bucket

Let's upload a file to the S3 bucket.
Open another terminal session and upload a file to S3.
You can manually upload a file from [Amazon Managed Console](https://s3.console.aws.amazon.com/s3) instead of execting below.

```bash
export BUCKETNAME=$(aws cloudformation describe-stacks --stack-name MountAmazonFSxOnAmazonEc2FSx --query "Stacks[0].Outputs[?OutputKey=='Bucket'].OutputValue" --output text)
echo '{"behavior":"import"}' > dummy_import.json
aws s3 cp dummy_import.json s3://$BUCKETNAME/dummy_import.json
```

Back to the ssh session and confirm the file existance.
There should be `dummy_import.json`.

```bash
sudo ls -la /mnt/fsx
```

### Export to S3 bucket

Let's create a file on `/mnt/fsx` and export it to the S3 bucket.
In the ssh session, execute below.

```bash
echo '{"behavior":"export"}' > dummy_export.json
sudo mv dummy_export.json /mnt/fsx

sudo lfs hsm_archive /mnt/fsx/dummy_export.json
sudo lfs hsm_action /mnt/fsx/dummy_export.json
```

Let's check the objects in the bucket.
There should be `dummy_export.json` that was exported from the instance.
Open another terminal and execute below.

```
export BUCKETNAME=$(aws cloudformation describe-stacks --stack-name MountAmazonFSxOnAmazonEc2FSx --query "Stacks[0].Outputs[?OutputKey=='Bucket'].OutputValue" --output text)
aws s3 ls s3://$BUCKETNAME
```

## Clean

Delete the stacks we deployed.
You can manually delete these stacks in [Amazon Managed Console](https://console.aws.amazon.com/cloudformation) instead of executing below.

```bash
aws cloudformation delete-stack --stack-name MountAmazonFSxOnAmazonEc2Instance
aws cloudformation delete-stack --stack-name MountAmazonFSxOnAmazonEc2FSx
aws cloudformation delete-stack --stack-name MountAmazonFSxOnAmazonEc2Vpc
```

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
