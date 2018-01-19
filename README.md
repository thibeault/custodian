# custodian
# Tag policy
# http://www.capitalone.io/cloud-custodian/docs/usecases/tagcompliance.html
# Lambda
# http://www.capitalone.io/cloud-custodian/docs/policy/lambda.html

## Good discussion can be found here
# https://gitter.im/capitalone/cloud-custodian

## Legacy create t2.micro test instance
aws ec2 run-instances --image-id ami-97785bed --count 1 --instance-type t2.micro --key-name aws-team --security-group-ids sg-18c9d474 --subnet-id subnet-7751335c --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=c7n_test},{Key=Owner,Value=ethibeault@shutterstock.com},{Key=Team,Value=ethibeault@shutterstock.com},{Key=CostCenter,Value=1528},{Key=BusinessUnits,Value=techops},{Key=Environment,Value=dev},{Key=Application,Value=c7n_test}]' 'ResourceType=volume,Tags=[{Key=Name,Value=c7n_test},{Key=Owner,Value=ethibeault@shutterstock.com},{Key=Team,Value=ethibeault@shutterstock.com},{Key=CostCenter,Value=1528},{Key=BusinessUnits,Value=techops},{Key=Environment,Value=dev},{Key=Application,Value=c7n_test}]'

### Architecture create t2.micro test instance
aws ec2 run-instances --image-id ami-97785bed --count 1 --instance-type t2.micro --key-name ethibeault-us-east-1 --security-group-ids sg-9a54beeb --subnet-id subnet-6644e33c --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=c7n_test},{Key=Owner,Value=ethibeault@shutterstock.com},{Key=Team,Value=ethibeault@shutterstock.com},{Key=CostCenter,Value=1528},{Key=BusinessUnits,Value=techops},{Key=Environment,Value=dev},{Key=Application,Value=c7n_test}]' 'ResourceType=volume,Tags=[{Key=Name,Value=c7n_test},{Key=Owner,Value=ethibeault@shutterstock.com},{Key=Team,Value=ethibeault@shutterstock.com},{Key=CostCenter,Value=1528},{Key=BusinessUnits,Value=techops},{Key=Environment,Value=dev},{Key=Application,Value=c7n_test}]'




# Vagrant notes
vagrant init ubuntu/trusty64
vagrant up
vagrant ssh


# Setup of the VM
sudo apt-get update
sudo apt-get install -y python-dev python-pip git
sudo apt-get install jq
sudo pip install awscli --upgrade
sudo pip install paramiko --upgrade
sudo pip install markupsafe --upgrade
sudo pip install --upgrade virtualenv
sudo pip install ansible --upgrade
sudo pip install boto3 --upgrade
sudo echo "complete -C '/usr/local/bin/aws_completer' aws" >> /home/vagrant/.bashrc
sudo ntpdate -s time.nist.gov

## create your roles
aws configure
aws iam create-role --role-name custodian-lambda-execution-role --assume-role-policy-document file://aws-roles/role-for-custodian.json

aws iam put-role-policy --role-name custodian-lambda-execution-role --policy-name custodian-lambda-permission-policy --policy-document file://aws-roles/lambda-execution-role-permissions.json


http://www.capitalone.io/cloud-custodian/docs/quickstart/index.html#install-cloud-custodian
----- initial - setup virtualenv and install c7n
virtualenv --python=python2 custodian
source custodian/bin/activate
(custodian) $ pip install c7n

----- back in - just source the virtualenv
cd /vagrant/custodian
source custodian/bin/activate

custodian validate policies/tag-compliance.yaml
custodian run --dryrun -s . --cache-period=0 policies/tag-compliance.yaml
custodian run -s . --cache-period=0 policies/tag-compliance.yaml
custodian schema ec2.actions.unmark


----
notify

How to decript a msg sent by c7n:
 cat result.txt | base64 -D > result.zlib
 printf "\x1f\x8b\x08\x00\x00\x00\x00\x00" | cat - result.zlib | gzip -dc > text.json


 how to have a policy send SNS
 - type: notify
   from: ethibeault@shutterstock.com
   to:
     - ethibeault@shutterstock.com
   subject: "Your instance is non-complient, it will be automaticaly stopped in 4 days"
   template: default
   transport:
     type: sns
     topic: arn:aws:sns:us-east-1:{account_id}:c7n_notification



misc:
- "tag:aws:autoscaling:groupName": absent
- "tag:c7n_tag_status": absent
- or:
  - "tag:Name": absent
  - "tag:Owner": absent
  - "tag:Team": absent
  - "tag:CostCenter": absent
  - "tag:BusinessUnits": absent
  - "tag:Environment": absent
  - "tag:Application": absent



############
policies

- ec2-auto-tag-user
desc:
    - Any instances created should get auto tag with CreatorId and CreatorName

- ec2-tag-compliance-mark
desc:
    - Look for any running instances that does not comply with our tagging policy
      Mark the instance for retirement in 14 days
      Notify owner of missing tags
req:
    - State == running
    - Not part of autoscaling group
    - Not already marked
    - "tag:Name": absent
    - "tag:Owner": absent
    - "tag:Team": absent
    - "tag:CostCenter": absent
    - "tag:BusinessUnits": absent
    - "tag:Environment": absent
    - "tag:Application": absent


- ec2-stop-notify-2
  - send second notification msg 7 days before
  - mark instance second notification initiated

- ec2-stop-notify-3
  - send final notification msg
  - remove second notification 1 day before
  - mark instance final notification initiated
