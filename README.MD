# S3 access points example repo

Example of using s3 access points to secure a multi-tenant s3 bucket. 
Tutorial available at [cloudsecuritymusings.com](https://www.cloudsecuritymusings.com)


Validating the CloudFormation template
```
aws cloudformation validate-template \
  --template-body file:///home/ec2-user/environment/s3-access-points/template.yaml
```

Creating the CloudFormation stack
```
aws cloudformation create-stack  \
  --stack-name s3-access-points-experimentation \
  --template-body file:///home/ec2-user/environment/s3-access-points/template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --on-failure DO_NOTHING \
  --parameters \
    ParameterKey=BucketName,ParameterValue=yummy-food \
    ParameterKey=IamUser,ParameterValue=myuser \
    ParameterKey=VpcId,ParameterValue=vpc-12345678
```

Updating the CloudFormation stack
```
aws cloudformation update-stack  \
  --stack-name s3-access-points-experimentation \
  --template-body file:///home/ec2-user/environment/s3-access-points/template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=BucketName,UsePreviousValue=true \
    ParameterKey=IamUser,UsePreviousValue=true \
    ParameterKey=VpcId,UsePreviousValue=true
```

Describing the stack
```
aws cloudformation describe-stacks --stack-name s3-access-points-experimentation
```

Getting stack outputs
```
CF_OUTPUTS=$(aws cloudformation describe-stacks \
  --stack-name s3-access-points-experimentation \
  --query "Stacks[0].Outputs")

CHEF_ROLE_ARN=$(echo ${CF_OUTPUTS} | \
  jq -r '.[] | select(.OutputKey=="ChefRoleArn").OutputValue')

CHEF_ACCESS_POINT_ARN=$(echo ${CF_OUTPUTS} | \
  jq -r '.[] | select(.OutputKey=="ChefAccessPointArn").OutputValue')

KETO_DIETER_ROLE_ARN=$(echo ${CF_OUTPUTS} | \
  jq -r '.[] | select(.OutputKey=="KetoDieterRoleArn").OutputValue')

KETO_DIETER_ACCESS_POINT_ARN=$(echo ${CF_OUTPUTS} | \
  jq -r '.[] | select(.OutputKey=="KetoDieterAccessPointArn").OutputValue')
  
OMNIVORE_ROLE_ARN=$(echo ${CF_OUTPUTS} | \
  jq -r '.[] | select(.OutputKey=="OmnivoreRoleArn").OutputValue')

OMNIVORE_ACCESS_POINT_ARN=$(echo ${CF_OUTPUTS} | \
  jq -r '.[] | select(.OutputKey=="OmnivoreAccessPointArn").OutputValue')
```

Assuming the chef role
```
ASSUME_ROLE_OUTPUT=$(aws sts assume-role \
  --role-arn ${CHEF_ROLE_ARN} \
  --role-session-name chef)     
  
export AWS_ACCESS_KEY_ID=$(echo $ASSUME_ROLE_OUTPUT | jq -r '.Credentials.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(echo $ASSUME_ROLE_OUTPUT | jq -r '.Credentials.SecretAccessKey')
export AWS_SESSION_TOKEN=$(echo $ASSUME_ROLE_OUTPUT | jq -r '.Credentials.SessionToken')

aws sts get-caller-identity
```

Putting and object few objects as chef
```
mkdir keto && \
  echo "steak" > keto/steak.txt && \
  echo "bacon" > keto/bacon.txt
mkdir standard && \
  echo "french fries" > standard/fries.txt && \
  echo "potato salad" > standard/potatosalad.txt

aws s3api put-object --bucket ${CHEF_ACCESS_POINT_ARN} --key keto/steak.txt \
  --body keto/steak.txt
aws s3api put-object --bucket ${CHEF_ACCESS_POINT_ARN} --key keto/bacon.txt \
  --body keto/bacon.txt
aws s3api put-object --bucket ${CHEF_ACCESS_POINT_ARN} --key standard/fries.txt \
  --body standard/fries.txt
aws s3api put-object --bucket ${CHEF_ACCESS_POINT_ARN} --key standard/potatosalad.txt \
  --body standard/potatosalad.txt
```

Attempt to get object like chef (which should fail):
```
aws s3api get-object --key standard/potatosalad.txt \
  --bucket ${CHEF_ACCESS_POINT_ARN} standard/potatosalad.txt
```


Get omnivore credentials
```
ASSUME_ROLE_OUTPUT=$(aws sts assume-role \
  --role-arn ${OMNIVORE_ROLE_ARN} \
  --role-session-name omnivore)     
  
export AWS_ACCESS_KEY_ID=$(echo $ASSUME_ROLE_OUTPUT | jq -r '.Credentials.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(echo $ASSUME_ROLE_OUTPUT | jq -r '.Credentials.SecretAccessKey')
export AWS_SESSION_TOKEN=$(echo $ASSUME_ROLE_OUTPUT | jq -r '.Credentials.SessionToken')

aws sts get-caller-identity
```

Get keto object and regular object
```
aws s3api get-object --key standard/potatosalad.txt \
  --bucket ${OMNIVORE_ACCESS_POINT_ARN} standard/potatosalad.txt

aws s3api get-object --key keto/bacon.txt \
  --bucket ${OMNIVORE_ACCESS_POINT_ARN} keto/bacon.txt
```

Things that fail an object
```
aws s3api put-object --bucket ${OMNIVORE_ACCESS_POINT_ARN} --key standard/potatosalad.txt \
  --body standard/potatosalad.txt

aws s3api get-object --key standard/potatosalad.txt \
  --bucket ${CHEF_ACCESS_POINT_ARN} standard/potatosalad.txt
```


Unset
```
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_SESSION_TOKEN
```

Get keto dieter credentials
```
ASSUME_ROLE_OUTPUT=$(aws sts assume-role \
  --role-arn ${KETO_DIETER_ROLE_ARN} \
  --role-session-name keto-dieter)     
  
export AWS_ACCESS_KEY_ID=$(echo $ASSUME_ROLE_OUTPUT | jq -r '.Credentials.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(echo $ASSUME_ROLE_OUTPUT | jq -r '.Credentials.SecretAccessKey')
export AWS_SESSION_TOKEN=$(echo $ASSUME_ROLE_OUTPUT | jq -r '.Credentials.SessionToken')

aws sts get-caller-identity
```

Get keto object
```
aws s3api get-object --key keto/bacon.txt \
  --bucket ${KETO_DIETER_ACCESS_POINT_ARN} keto/bacon.txt
```

Attempt to get another object
```
aws s3api get-object --key standard/potatosalad.txt \
  --bucket ${KETO_DIETER_ACCESS_POINT_ARN} standard/potatosalad.txt
```
