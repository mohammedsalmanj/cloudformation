# Define Parameters (Input Values)
Parameters:
  # Parameter for specifying the service name
  NameOfService:
    Description: "Service name"
    Type: String

  # Parameter for specifying the CIDR block for Security Group Ingress
  AllowedCidrIp:
    Description: "CIDR block for Security Group Ingress."
    Type: String
    Default: 0.0.0.0/0

  # Parameter for specifying the instance type with allowed values
  InstanceTypeParameter:
    Description: "Enter t2.micro, m1.small, or m1.large. Default is t2.micro."
    Type: String
    Default: t2.micro
    AllowedValues:
      - t2.micro
      - m1.small
      - m1.large

  # Parameter for specifying the EC2 login key name
  KeyName:
    Description: "Name of EC2 login"
    Type: AWS::EC2::KeyPair::KeyName

# Define AMI mappings based on AWS region
Mappings:
  # Mapping region names to AMI IDs
  AmiRegionMap:
    us-east-1:
      AMI: "ami-0fa1ca9559f1892ec"
    us-east-2:
      AMI: "ami-09f85f3aaae282910"
    # Add more regions and corresponding AMI IDs as needed

# Define AWS Resources
Resources:
  # Define an EC2 Instance
  WebServer:
    Type: AWS::EC2::Instance
    Metadata:
      # Metadata for initializing the EC2 instance
      AWS::CloudFormation::Init:
        config:
          packages:
            # Install necessary packages using yum
            yum:
              php: []
              httpd: []
              wget: []
              unzip: []
              git: []
          commands:
            # Execute a test command to download and unzip a template
            test:
              command: "wget https://www.tooplate.com/zip-templates/2117_infinite_loop.zip && unzip -o 2117_infinite_loop.zip && cp -r 2117_infinite_loop/* /var/www/html/"
          files:
            # Create a hello.html file in /var/www/html
            /var/www/html/hello.html:
              content: |
                <!DOCTYPE html>
                <html>
                <body>
                <h1>Welcome to CloudFormation.</h1>
                <p>This site is deployed by CloudFormation.</p>
                </body>
                </html>
          services:
            sysvinit:
              httpd:
                enabled: true
                ensureRunning: true
    Properties:
      # EC2 instance properties
      InstanceType: !Ref InstanceTypeParameter
      KeyName: !Ref KeyName
      ImageId: !FindInMap
        - AmiRegionMap
        - !Ref "AWS::Region"
        - AMI
      Tags:
        - Key: "Name"
          Value: !Ref NameOfService
    
      # Attach Security Groups to the EC2 Instance
      SecurityGroupIds:
        - !Ref MySecurityGroup
    
      UserData:
        # User data script to install and configure software on the EC2 instance
        "Fn::Base64": !Sub |
          #!/bin/bash -xe            
          # Ensure AWS CFN Bootstrap is the latest
          yum install -y aws-cfn-bootstrap
          # Install the files and packages from the metadata
          /opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource WebServer --region ${AWS::Region}
          /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource WebServer --region ${AWS::Region}

  # Define an EC2 Security Group
  MySecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: "Allow HTTP traffic"
      
      # Define Ingress Rules for the Security Group
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: !Ref AllowedCidrIp

# Define Outputs
Outputs:
  # Output the public DNS name of the EC2 instance
  PrintSomeInfo:
    Value: !GetAtt
      - WebServer
      - PublicDnsName
