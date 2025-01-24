Title: Quick and Easy Clojure on AWS Lambda Part 2
Date: 2025-01-24
Tags: clojure

This follows on from my [previous article](lambda1.html) which described how to get a simple Clojure Ring application running on AWS Lambda. This article shows how to connect it to a database.

The accompanying code is [here](https://github.com/paulbutcher/example-lambda-app2).

## CloudFormation

AWS SAM provides direct support for DynamoDB, but not for more traditional databases like PostgreSQL, so that means dropping into CloudFormation. This is, sadly, rather wordy, because we'll need to configure all the neccessary AWS machinery (including setting up a VPC, security group, and database credentials) ourselves, but it's mostly standard boilerplate which is pretty well documented:

* [Giving Lambda functions access to resources in an Amazon VPC](https://docs.aws.amazon.com/lambda/latest/dg/configuration-vpc.html).
* [Creating a Secrets Manager secret for a master password](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-rds-dbinstance.html#aws-resource-rds-dbinstance--examples--Creating_a_Secrets_Manager_secret_for_a_master_password).

You can see the full CloudFormation template [here](https://github.com/paulbutcher/example-lambda-app2/blob/main/template.yaml). We'll explain the various sections in more detail below.

### VPC

To avoid having to make our database publicly visible, we're going to put both our Lambda function and database in a shared VPC. Here's how we create that VPC:

```yaml
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
```

We also need to define a couple of subnets (RDS requires at least two subnets, in two different avaialability zones):

```yaml
  Subnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, Fn::GetAZs: !Ref "AWS::Region"]
  Subnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: !Select [1, Fn::GetAZs: !Ref "AWS::Region"]
```

This is using a little CloudFormation magic to select the first two availability zones in whichever region we're deploying to. You could just as easily hardcode the availability zones if you prefer.

And we need a security group which allows things within the VPC to access Postgres:

```yaml
  SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: !Sub "Security group for ${AWS::StackName}"
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 5432
          ToPort: 5432
          CidrIp: !GetAtt VPC.CidrBlock
```

We put our Lambda function in the VPC we've created by adding the following to its `Properties`:

```yaml
      VpcConfig:
        SecurityGroupIds: [!Ref SecurityGroup]
        SubnetIds: [!Ref Subnet1, !Ref Subnet2]
      Policies: [AWSLambdaVPCAccessExecutionRole]
```

### Database

We now have everything we need to create a database instance:

```yaml
  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: !Sub "DBSubnet group for ${AWS::StackName}"
      SubnetIds: [!Ref Subnet1, !Ref Subnet2]
  Database:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceClass: db.t4g.micro
      Engine: postgres
      EngineVersion: 14.15
      DBName: example_lambda_app
      AllocatedStorage: 20
      StorageEncrypted: true
      ManageMasterUserPassword: true
      MasterUsername: postgres
      KmsKeyId: !Ref DatabaseKey
      VPCSecurityGroups: [!Ref SecurityGroup]
      DBSubnetGroupName: !Ref DBSubnetGroup
```

Most of this is pretty obvious: we're creating an RDS database running on a `db.t4g.micro` instance, with 20GB of storage, encrypted at rest, and with a master user called `postgres`. We're adding it to the security group we created earlier, and letting it know about the subnets we created via a `DBSubnetGroup`.

### Key Management

We've asked RDS to manage the database password for us (`ManageMasterUserPassword`) and store the credentials in a Secrets Manager (KMS) secret. Here's how we create that secret:

```yaml
  DatabaseKey:
    Type: AWS::KMS::Key
    Properties:
      Description: DatabaseKey
      EnableKeyRotation: false
      KeyPolicy:
        Version: 2012-10-17
        Id: !Sub "key-${AWS::StackName}"
        Statement:
          - Effect: Allow
            Principal:
              AWS: !Sub "arn:${AWS::Partition}:iam::${AWS::AccountId}:root"
            Action: ["kms:*"]
            Resource: "*"
```

I've chosen to disable key rotation because I'll be passing the key to the Lambda function as an environment variable. An alternative would be to modify the Lambda function to use the Secrets Manager API to retrieve the password, but I wanted to keep the code as simple as possible. The rest is simple boilerplate taken from the article mentioned above.

To use this secret in our Lambda function, we add the following to the function's `Properties`:

```yaml
      Environment:
        Variables:
          DB_HOST: !GetAtt Database.Endpoint.Address
          DB_PASSWORD: !Sub "{{resolve:secretsmanager:${Database.MasterUserSecret.SecretArn}:SecretString:password}}"
```

### Deployment

Deploying is exactly the same as before: build the uberjar and then `sam deploy`. The first time you do this it'll take a while because it's creating the database instance, but subsequent deployments will be much quicker.
