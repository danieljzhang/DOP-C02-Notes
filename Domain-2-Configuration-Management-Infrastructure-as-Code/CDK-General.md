# AWS CDK (Cloud Development Kit) - DOP-C02 Exam Notes

## 1. Overview

**AWS CDK** is an open-source software development framework for defining cloud infrastructure using familiar programming languages like TypeScript, Python, Java, C#, and Go.

### Key Characteristics
- **Programming languages** - Use familiar languages instead of JSON/YAML
- **Higher-level abstractions** - Constructs provide sensible defaults
- **Type safety** - Compile-time error checking
- **IDE support** - IntelliSense, refactoring, debugging
- **Reusable components** - Share constructs across projects
- **CloudFormation integration** - Synthesizes to CloudFormation templates
- **AWS service integration** - Native support for all AWS services

### What Problem Does It Solve?
- Eliminates verbose CloudFormation JSON/YAML syntax
- Provides programming language benefits (loops, conditions, functions)
- Enables infrastructure testing with standard testing frameworks
- Facilitates code reuse through object-oriented programming
- Supports complex logic and data transformations
- Integrates with existing development workflows

---

## 2. Core Concepts

### Constructs
```typescript
// Level 1 (L1) - Direct CloudFormation mapping
import { CfnBucket } from 'aws-cdk-lib/aws-s3';

const cfnBucket = new CfnBucket(this, 'MyBucket', {
  bucketName: 'my-bucket-name',
  versioningConfiguration: {
    status: 'Enabled'
  }
});

// Level 2 (L2) - Higher-level with sensible defaults
import { Bucket, BucketEncryption } from 'aws-cdk-lib/aws-s3';

const bucket = new Bucket(this, 'MyBucket', {
  bucketName: 'my-bucket-name',
  versioned: true,
  encryption: BucketEncryption.S3_MANAGED,
  removalPolicy: RemovalPolicy.DESTROY
});

// Level 3 (L3) - Patterns combining multiple resources
import { ApplicationLoadBalancedFargateService } from 'aws-cdk-lib/aws-ecs-patterns';

const service = new ApplicationLoadBalancedFargateService(this, 'MyService', {
  taskImageOptions: {
    image: ContainerImage.fromRegistry('nginx'),
    containerPort: 80
  },
  memoryLimitMiB: 512,
  cpu: 256,
  desiredCount: 2
});
```

### Stacks and Apps
```typescript
// app.ts
import { App } from 'aws-cdk-lib';
import { VpcStack } from './vpc-stack';
import { DatabaseStack } from './database-stack';
import { WebStack } from './web-stack';

const app = new App();

const vpcStack = new VpcStack(app, 'VpcStack', {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: process.env.CDK_DEFAULT_REGION
  }
});

const dbStack = new DatabaseStack(app, 'DatabaseStack', {
  vpc: vpcStack.vpc,
  env: vpcStack.env
});

const webStack = new WebStack(app, 'WebStack', {
  vpc: vpcStack.vpc,
  database: dbStack.database,
  env: vpcStack.env
});

// Add dependencies
dbStack.addDependency(vpcStack);
webStack.addDependency(dbStack);
```

### Stack Definition
```typescript
// vpc-stack.ts
import { Stack, StackProps } from 'aws-cdk-lib';
import { Vpc, SubnetType } from 'aws-cdk-lib/aws-ec2';
import { Construct } from 'constructs';

export interface VpcStackProps extends StackProps {
  cidrBlock?: string;
}

export class VpcStack extends Stack {
  public readonly vpc: Vpc;

  constructor(scope: Construct, id: string, props?: VpcStackProps) {
    super(scope, id, props);

    this.vpc = new Vpc(this, 'MainVpc', {
      cidr: props?.cidrBlock || '10.0.0.0/16',
      maxAzs: 3,
      subnetConfiguration: [
        {
          cidrMask: 24,
          name: 'Public',
          subnetType: SubnetType.PUBLIC
        },
        {
          cidrMask: 24,
          name: 'Private',
          subnetType: SubnetType.PRIVATE_WITH_EGRESS
        },
        {
          cidrMask: 28,
          name: 'Database',
          subnetType: SubnetType.PRIVATE_ISOLATED
        }
      ],
      natGateways: 1
    });
  }
}
```

---

## 3. CDK Pipelines

### Self-Mutating Pipeline
```typescript
// pipeline-stack.ts
import { Stack, StackProps } from 'aws-cdk-lib';
import { CodePipeline, CodePipelineSource, ShellStep } from 'aws-cdk-lib/pipelines';
import { Repository } from 'aws-cdk-lib/aws-codecommit';

export class PipelineStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const repo = Repository.fromRepositoryName(this, 'Repo', 'my-cdk-app');

    const pipeline = new CodePipeline(this, 'Pipeline', {
      pipelineName: 'MyAppPipeline',
      synth: new ShellStep('Synth', {
        input: CodePipelineSource.codeCommit(repo, 'main'),
        commands: [
          'npm ci',
          'npm run build',
          'npx cdk synth'
        ]
      })
    });

    // Add application stages
    const devStage = new MyAppStage(this, 'Dev', {
      env: { account: '111111111111', region: 'us-east-1' }
    });
    
    const prodStage = new MyAppStage(this, 'Prod', {
      env: { account: '222222222222', region: 'us-east-1' }
    });

    pipeline.addStage(devStage, {
      pre: [
        new ShellStep('UnitTests', {
          commands: ['npm test']
        })
      ],
      post: [
        new ShellStep('IntegrationTests', {
          commands: ['npm run test:integration']
        })
      ]
    });

    pipeline.addStage(prodStage, {
      pre: [
        new ManualApprovalStep('PromoteToProd')
      ]
    });
  }
}
```

### Application Stage
```typescript
// app-stage.ts
import { Stage, StageProps } from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { VpcStack } from './vpc-stack';
import { DatabaseStack } from './database-stack';
import { WebStack } from './web-stack';

export class MyAppStage extends Stage {
  constructor(scope: Construct, id: string, props?: StageProps) {
    super(scope, id, props);

    const vpcStack = new VpcStack(this, 'VpcStack');
    const dbStack = new DatabaseStack(this, 'DatabaseStack', {
      vpc: vpcStack.vpc
    });
    const webStack = new WebStack(this, 'WebStack', {
      vpc: vpcStack.vpc,
      database: dbStack.database
    });
  }
}
```

---

## 4. Testing CDK Applications

### Unit Testing
```typescript
// test/vpc-stack.test.ts
import { Template } from 'aws-cdk-lib/assertions';
import { App } from 'aws-cdk-lib';
import { VpcStack } from '../lib/vpc-stack';

describe('VpcStack', () => {
  test('creates VPC with correct configuration', () => {
    const app = new App();
    const stack = new VpcStack(app, 'TestVpcStack');
    const template = Template.fromStack(stack);

    // Assert VPC exists
    template.hasResourceProperties('AWS::EC2::VPC', {
      CidrBlock: '10.0.0.0/16',
      EnableDnsHostnames: true,
      EnableDnsSupport: true
    });

    // Assert subnets are created
    template.resourceCountIs('AWS::EC2::Subnet', 6); // 3 AZs * 2 subnet types
    
    // Assert NAT Gateway exists
    template.resourceCountIs('AWS::EC2::NatGateway', 1);
  });

  test('allows custom CIDR block', () => {
    const app = new App();
    const stack = new VpcStack(app, 'TestVpcStack', {
      cidrBlock: '172.16.0.0/16'
    });
    const template = Template.fromStack(stack);

    template.hasResourceProperties('AWS::EC2::VPC', {
      CidrBlock: '172.16.0.0/16'
    });
  });
});
```

### Integration Testing
```typescript
// test/integration.test.ts
import { IntegTest } from '@aws-cdk/integ-tests-alpha';
import { App } from 'aws-cdk-lib';
import { VpcStack } from '../lib/vpc-stack';

const app = new App();
const stack = new VpcStack(app, 'IntegTestVpcStack');

new IntegTest(app, 'VpcIntegTest', {
  testCases: [stack],
  diffAssets: true,
  stackUpdateWorkflow: true
});
```

---

## 5. Custom Constructs

### Reusable Construct
```typescript
// lib/constructs/web-server.ts
import { Construct } from 'constructs';
import { Instance, InstanceType, MachineImage, Vpc, SecurityGroup } from 'aws-cdk-lib/aws-ec2';
import { Role, ServicePrincipal, ManagedPolicy } from 'aws-cdk-lib/aws-iam';

export interface WebServerProps {
  vpc: Vpc;
  instanceType?: InstanceType;
  keyName?: string;
}

export class WebServer extends Construct {
  public readonly instance: Instance;
  public readonly securityGroup: SecurityGroup;

  constructor(scope: Construct, id: string, props: WebServerProps) {
    super(scope, id);

    // Security Group
    this.securityGroup = new SecurityGroup(this, 'SecurityGroup', {
      vpc: props.vpc,
      description: 'Security group for web server',
      allowAllOutbound: true
    });

    this.securityGroup.addIngressRule(
      Peer.anyIpv4(),
      Port.tcp(80),
      'Allow HTTP traffic'
    );

    this.securityGroup.addIngressRule(
      Peer.anyIpv4(),
      Port.tcp(443),
      'Allow HTTPS traffic'
    );

    // IAM Role
    const role = new Role(this, 'Role', {
      assumedBy: new ServicePrincipal('ec2.amazonaws.com'),
      managedPolicies: [
        ManagedPolicy.fromAwsManagedPolicyName('AmazonSSMManagedInstanceCore')
      ]
    });

    // EC2 Instance
    this.instance = new Instance(this, 'Instance', {
      vpc: props.vpc,
      instanceType: props.instanceType || InstanceType.of(InstanceClass.T3, InstanceSize.MICRO),
      machineImage: MachineImage.latestAmazonLinux(),
      securityGroup: this.securityGroup,
      role: role,
      keyName: props.keyName,
      userData: UserData.forLinux()
    });

    // User data script
    this.instance.userData.addCommands(
      'yum update -y',
      'yum install -y httpd',
      'systemctl start httpd',
      'systemctl enable httpd',
      'echo "<h1>Hello from CDK!</h1>" > /var/www/html/index.html'
    );
  }
}
```

### Using Custom Construct
```typescript
// lib/web-stack.ts
import { Stack, StackProps } from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { WebServer } from './constructs/web-server';

export interface WebStackProps extends StackProps {
  vpc: Vpc;
}

export class WebStack extends Stack {
  constructor(scope: Construct, id: string, props: WebStackProps) {
    super(scope, id, props);

    const webServer = new WebServer(this, 'WebServer', {
      vpc: props.vpc,
      instanceType: InstanceType.of(InstanceClass.T3, InstanceSize.SMALL)
    });

    new CfnOutput(this, 'WebServerInstanceId', {
      value: webServer.instance.instanceId,
      description: 'Web server instance ID'
    });
  }
}
```

---

## 6. Environment and Context

### Environment Configuration
```typescript
// cdk.json
{
  "app": "npx ts-node --prefer-ts-exts bin/app.ts",
  "context": {
    "@aws-cdk/core:enableStackNameDuplicates": true,
    "aws-cdk:enableDiffNoFail": true,
    "@aws-cdk/core:stackRelativeExports": true
  }
}

// Environment-specific configuration
const app = new App();

// Development environment
new MyAppStack(app, 'MyApp-Dev', {
  env: {
    account: '111111111111',
    region: 'us-east-1'
  },
  tags: {
    Environment: 'Development',
    CostCenter: 'Engineering'
  }
});

// Production environment
new MyAppStack(app, 'MyApp-Prod', {
  env: {
    account: '222222222222',
    region: 'us-east-1'
  },
  tags: {
    Environment: 'Production',
    CostCenter: 'Operations'
  }
});
```

### Context Values
```typescript
// Using context for configuration
export class MyStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const instanceType = this.node.tryGetContext('instanceType') || 't3.micro';
    const enableMonitoring = this.node.tryGetContext('enableMonitoring') === 'true';
    
    const instance = new Instance(this, 'Instance', {
      instanceType: new InstanceType(instanceType),
      // ... other properties
    });

    if (enableMonitoring) {
      // Add CloudWatch monitoring
    }
  }
}

// Set context via CLI
// cdk deploy -c instanceType=t3.small -c enableMonitoring=true
```

---

## 7. CDK CLI Commands

```bash
# Initialize new CDK project
cdk init app --language typescript
cdk init app --language python

# Install dependencies
npm install
pip install -r requirements.txt

# Synthesize CloudFormation template
cdk synth
cdk synth MyStack

# Deploy stacks
cdk deploy
cdk deploy MyStack
cdk deploy --all

# Destroy stacks
cdk destroy
cdk destroy MyStack

# Diff changes
cdk diff
cdk diff MyStack

# List stacks
cdk list

# Bootstrap CDK environment
cdk bootstrap
cdk bootstrap aws://123456789012/us-east-1

# Watch for changes (CDK v2)
cdk watch
cdk watch MyStack
```

---

## 8. CDK vs CloudFormation vs Terraform

| Feature | CDK | CloudFormation | Terraform |
|---------|-----|----------------|-----------|
| **Language** | Programming languages | JSON/YAML | HCL |
| **Type Safety** | Yes | No | Limited |
| **IDE Support** | Full | Limited | Good |
| **Testing** | Unit/Integration | Limited | Limited |
| **Reusability** | High (OOP) | Medium (nested stacks) | High (modules) |
| **Learning Curve** | Moderate | Steep | Moderate |
| **Multi-cloud** | AWS only | AWS only | Yes |
| **State Management** | CloudFormation | AWS managed | External |

---

## 9. Common Exam Scenarios

### Scenario 1: Multi-Stack Application
```typescript
// Cross-stack references
export class DatabaseStack extends Stack {
  public readonly database: DatabaseInstance;
  
  constructor(scope: Construct, id: string, props: StackProps) {
    super(scope, id, props);
    
    this.database = new DatabaseInstance(this, 'Database', {
      // configuration
    });
  }
}

export class WebStack extends Stack {
  constructor(scope: Construct, id: string, props: WebStackProps) {
    super(scope, id, props);
    
    // Reference database from another stack
    const dbEndpoint = props.database.instanceEndpoint.hostname;
    
    // Use in application configuration
  }
}
```

### Scenario 2: Conditional Resources
```typescript
export class ConditionalStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const environment = this.node.tryGetContext('environment') || 'dev';
    
    // Conditional resource creation
    if (environment === 'prod') {
      new DatabaseInstance(this, 'Database', {
        instanceType: InstanceType.of(InstanceClass.R5, InstanceSize.LARGE),
        multiAz: true,
        backupRetention: Duration.days(30)
      });
    } else {
      new DatabaseInstance(this, 'Database', {
        instanceType: InstanceType.of(InstanceClass.T3, InstanceSize.MICRO),
        multiAz: false,
        backupRetention: Duration.days(7)
      });
    }
  }
}
```

### Scenario 3: Custom Resource
```typescript
import { CustomResource, CustomResourceProvider } from 'aws-cdk-lib';
import { Function, Runtime, Code } from 'aws-cdk-lib/aws-lambda';

export class CustomResourceStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    // Lambda function for custom resource
    const customResourceLambda = new Function(this, 'CustomResourceLambda', {
      runtime: Runtime.PYTHON_3_9,
      handler: 'index.handler',
      code: Code.fromInline(`
import boto3
import json

def handler(event, context):
    if event['RequestType'] == 'Create':
        # Custom logic here
        return {
            'PhysicalResourceId': 'custom-resource-id',
            'Data': {'Result': 'Success'}
        }
    return {'PhysicalResourceId': event['PhysicalResourceId']}
      `)
    });

    // Custom resource
    const customResource = new CustomResource(this, 'CustomResource', {
      serviceToken: customResourceLambda.functionArn,
      properties: {
        CustomProperty: 'CustomValue'
      }
    });
  }
}
```

---

## 10. Exam Tips

- **Understand construct levels** - L1, L2, L3 constructs and when to use each
- **Master stack dependencies** - Cross-stack references and deployment order
- **Know CDK Pipelines** - Self-mutating pipelines and multi-environment deployment
- **Practice testing** - Unit tests with assertions, integration tests
- **Understand synthesis** - How CDK generates CloudFormation templates
- **Learn custom constructs** - Creating reusable components
- **Know CLI commands** - Bootstrap, deploy, diff, destroy operations
- **Understand context** - Environment-specific configuration
- **Practice with patterns** - Common architectural patterns in CDK
- **Compare with alternatives** - When to use CDK vs CloudFormation vs Terraform