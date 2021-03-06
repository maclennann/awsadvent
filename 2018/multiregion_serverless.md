# Multi-region Serverless APIs: I've got a fever and the only cure is fewer servers

## Meet SAM

Let's talk about the hottest thing in computers for the past few years. No, not Machine Learning. No, not
Kubernetes. No, not big data. Fine, _one of_ the hottest things in computers. Right, serverless!

It's still an emerging and quickly changing field, but I'd like to take some time to demonstrate how easy
it is to make scalable and reliable multi-region APIs using just a few serverless tools and services.

It's actually deceptively simple. Well, for a "blog post"-level application, anyway.

We're going to be managing this application using the wonderful [AWS Serverless Application Model](https://github.com/awslabs/serverless-application-model) (SAM)
and [SAM CLI](https://github.com/awslabs/aws-sam-cli). 
Far and away the easiest way I have ever used for creating and deploying serverless 
applications. And, in keeping with contemporary practices, it even has a cute little animal mascot.

SAM is a feature of CloudFormation that provides a handful of short-hand resources that get expanded out
to their equivalent long-hand CloudFormation resources upon ChangeSet calculation. You can also drop down into
regular CloudFormation whenever you need to in order to manage resources and configurations not covered
by SAM.

The SAM CLI is a local CLI application for developing, testing, and deploying your SAM applications.
It uses Docker under the hood to provide as close to a Lambda execution environment as possible and even
allows you to run your APIs locally in an APIGateway-like environment. It's pretty great, IMO.

So if you're following along, go ahead and [install Docker and the SAM CLI](https://github.com/awslabs/aws-sam-cli/blob/develop/docs/getting_started.rst) and we can get started.

## The SAMple App
Once that's installed, let's generate a sample application so we can see what it's all about. If you're
following along on the terminal, you can run `sam init -n hello-sam -r nodejs8.10` to generate a sample
node app called `hello-sam`. You can also see the output in the [hello-sam-1 folder](/2018/hello-sam-1)
in the linked repo if you aren't at a terminal and just want to read along.

The first thing to notice is the `README.md` that is full of a huge amount of information about the repo.
For the sake of brevity, I'm going to leave learning the basics of SAM and the repo structure you're
looking at as a bit of an exercise for the reader. The README and linked documentation can tell you anything
you need to know.

The important thing to know is that `hello_world/` contains the code and `template.yaml` contains a special
SAM-flavored CloudFormation template that controls the application. Take some time to familiarize yourself
with it if you want to.

## SAM Local

So what can SAM do, other than give us very short CFN templates? Well the SAM CLI can do a lot to help you in 
your local development process. Let's try it out.

Step 0 is to install your npm dependencies so your function can execute:

```
$ cd hello_world/
$ npm installadded 34 packages from 460 contributors and audited 44 packages in 1.192s
found 0 vulnerabilities

$ cd ..
```

Alright, now let's have some fun.

There are a few local invocation commands that I won't cover here, because we're making an API. And the 
real magic with the CLI is that you can actually run your API locally with `sam local start-api`.
This will inspect your template, identify your API schema, start a local API Gateway, and mount your functions
at the correct paths. It's by no means a perfect replica of running in production, but it actually does a
surprisingly great job.

When we start the API, it will mount our function at `/hello`, following the path specified in the `Events`
attribute of the resource.

```
$ sam local start-api
2018-11-28 10:08:21 Mounting HelloWorldFunction at http://127.0.0.1:3000/hello [GET]
2018-11-28 10:08:21 You can now browse to the above endpoints to invoke your functions. You do not need to restart/reload SAM CLI while working on your functions changes will be reflected instantly/automatically. You only need to restart SAM CLI if you update your AWS SAM template
2018-11-28 10:08:21  * Running on http://127.0.0.1:3000/ (Press CTRL+C to quit)
```

Now you can go ahead and curl against the advertised port and path to execute your function.

```
$ curl localhost:3000/hello
{"message":"hello world","location":"[redacted]"}
```

Then on the backend, you'll see it executing your function in Docker:

```
2018-11-28 10:12:12 Invoking app.lambda_handler (nodejs8.10)
2018-11-28 10:12:12 Starting new HTTP connection (1): 169.254.169.254

Fetching lambci/lambda:nodejs8.10 Docker container image......
2018-11-28 10:12:15 Mounting /home/nmaclennan/src/github.com/maclennann/awsadvent/2018/hello-sam-1/hello_world as /var/task:ro inside runtime container
START RequestId: 7dbba90c-fe1f-160b-c409-59e8d56cf6d6 Version: $LATEST
END RequestId: 7dbba90c-fe1f-160b-c409-59e8d56cf6d6
REPORT RequestId: 7dbba90c-fe1f-160b-c409-59e8d56cf6d6  Duration: 316.61 ms     Billed Duration: 400 ms Memory Size: 128 MB     Max Memory Used: 33 MB
2018-11-28 10:12:16 No Content-Type given. Defaulting to 'application/json'.
2018-11-28 10:12:16 127.0.0.1 - - [28/Nov/2018 10:12:16] "GET /hello HTTP/1.1" 200 -
```

You can change your code at-will and the next invocation will pick it up. You can also attach a debugger to
the process if you aren't a "debug statement developer."

Want to try to deploy it? I guess we might as well - it's easy enough. The only pre-requisite is that
we need an S3 bucket for our uploaded code artifact. So go ahead and make that - call it whatever you like.

```
$ aws s3 mb s3://its-my-bucket-and-ill-deploy-if-i-want-to
make_bucket: its-my-bucket-and-ill-deploy-if-i-want-to
```

Now, we'll run a `sam package`. This will bundle up the code for all of your functions and upload
it to S3. It'll spit out a rendered "deployment template" that has the local `CodeUri`s swapped out for S3 URLs.

```
$ sam package --template-file template.yaml --output-template-file deploy-template.yaml --s3-bucket its-my-bucket-and-ill-deploy-if-i-want-to
Uploading to 3dfb2bd4c72dad68cc4074476b0b40d3  958016 / 958016.0  (100.00%)
Successfully packaged artifacts and wrote output template to file deploy-template.yaml.
Execute the following command to deploy the packaged template
aws cloudformation deploy --template-file /home/nmaclennan/src/github.com/maclennann/awsadvent/2018/hello-sam-1/deploy-template.yaml --stack-name <YOUR STACK NAME>
```

If you check out `deploy-template.yaml`, you should see...very few remarkable differences.
Maybe some of the properties have been re-ordered or blank lines removed. But the only real
difference you should see is that the relative CodeUrl of `CodeUri: hello_world/` for your
function has been resolved to an S3 URL for deployment.

Now let's go ahead and deploy it!

```
$ sam deploy --template-file deploy-template.yaml --stack-name hello-sam --capabilities CAPABILITY_IAM --region us-east-1

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - hello-sam
```

Lastly, let's find the URL for our API so we can try it out:

```
$ aws cloudformation describe-stacks --stack-name hello-sam --query 'Stacks[].Outputs'
[...snip...]
        {
            "Description": "API Gateway endpoint URL for Prod stage for Hello World function",
            "OutputKey": "HelloWorldApi",
            "OutputValue": "https://[redacted].execute-api.us-east-1.amazonaws.com/Prod/hello/"
        },
[...snip...]
```

Cool, let's try it:

```
$ curl https://[redacted].execute-api.us-east-1.amazonaws.com/Prod/hello/
{"message":"hello world","location":"[redacted]"}
```

Nice work! Now that you know how SAM works, let's make it do some real work for us.

## State and Data

We're planning on taking this multi-region by the end of this post. Deploying
a multi-region application with no state or data is both easy and boring. Let's do something 
interesting and add some data to our application. For the purposes of this post, let's do something
simple like storing a per-IP hit counter in DynamoDB.

We'll go through the steps below, but if you want to jump right to done, check out
[the hello-sam-2 folder](/2018/hello-sam-2) in this repository.

SAM offers a `SimpleTable` resource that creates a very simple DynamoDB table. This is
technically fine for our use-case now, but we'll need to be able to enable Table Streams in
the future to go multi-region. So we'll need to use the regular `DynamoDB::Table` resource:

```
HitsTable:
  Type: AWS::DynamoDB::Table
  Properties:
    TableName: HitsTable
    AttributeDefinitions:
      - AttributeName: ipAddress
        AttributeType: S
    KeySchema:
      - AttributeName: ipAddress
        KeyType: HASH
    ProvisionedThroughput:
      ReadCapacityUnits: 5
      WriteCapacityUnits: 5
    SSESpecification:
      SSEEnabled: true
    StreamSpecification:
      StreamViewType: NEW_AND_OLD_IMAGES
```

We can use environment variables to let our functions know what our table is named instead
of hard-coding it in code. Let's add an environment variable up in the `Globals`
section. This ensures that any functions we may add in the future automatically have access
to this as well.

Change your `Globals` section to look like:

```
Globals:
  Function:
    Timeout: 3
    Environment:
      Variables:
        HITS_TABLE_NAME: !Ref HitsTable
```

Lastly, we'll need to give our existing function access to update and read items from the table.
We'll do that by setting the `Policies` attribute of the resource, which turns into the execution role.
We'll give the function `UpdateItem` and `GetItem`. When you're done, the resource should look like:

```
  HelloWorldFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: hello_world/
      Handler: app.lambda_handler
      Runtime: nodejs8.10
      Policies:
        - AWSXrayWriteOnlyAccess
        - Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - 'dynamodb:GetItem'
                - 'dynamodb:UpdateItem'
              Resource: !GetAtt HitsTable.Arn
      Events:
        HelloWorld:
          Type: Api
          Properties:
            Path: /hello
            Method: get
```

Now let's have our function start using the table. Crack open `hello_world/app.js`
and replace the content with:

```
const {DynamoDB} = require('aws-sdk'),
  dynamo = new DynamoDB();

exports.lambda_handler = async (event) => {
  const sourceIp = event.requestContext.identity.sourceIp;
  
  let resp = await dynamo.updateItem({
    TableName: process.env.HITS_TABLE_NAME,
    Key: {
      ipAddress: { S: sourceIp}
    },
    UpdateExpression: 'ADD hits :h',
    ExpressionAttributeValues: {
      ':h': { N: '1' }
    },
    ReturnValues: 'ALL_NEW'
  }).promise();

  return {
    statusCode: 200,
    body: JSON.stringify({
      message: `hello from ${process.env.AWS_REGION}, ${sourceIp}`,
      visits: resp.Attributes.hits.N
    })
  };
};
```

On each request, our function will read the requester's IP from the event, increment a counter 
for that IP in the DynamoDB table, and return the total number of hits for that IP to the user.

This'll probably work in production, but we want to be dilligent and test it because we're responsible,
right? Normally, I'd recommend you spin up a [DynamoDB-local Docker container](https://hub.docker.com/r/amazon/dynamodb-local/),
but to keep things simple for the purposes of this post, let's create a "local dev" table in our
AWS account called `HitsTableLocal`.

```
$ aws dynamodb create-table --table-name HitsTableLocal --attribute-definitions AttributeName=ipAddress,AttributeType=S --key-schema AttributeName=ipAddress,KeyType=HASH --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5
```

And now let's update our function to use that table when we're executing locally. We can use the `AWS_SAM_LOCAL`
environment variable to determine if we're running locally or not. Toss this at the top of your `app.js` to select
that table when running locally:

```
if(process.env.AWS_SAM_LOCAL) {
  console.log("RUNNING LOCALLY")
  process.env.HITS_TABLE_NAME='HitsTableLocal';
}
```

Now let's give it a shot! Fire up the app with `sam local start-api` and let's do some curls.

```
$ curl localhost:3000/hello
{"message":"hello from us-east-1, 127.0.0.1","visits":"1"}
$ curl localhost:3000/hello
{"message":"hello from us-east-1, 127.0.0.1","visits":"2"}
$ curl localhost:3000/hello
{"message":"hello from us-east-1, 127.0.0.1","visits":"3"}
```

Nice! Now let's deploy it and try it out for real.

```
$ AWS_PROFILE=personal sam package --template-file template.yaml --output-template-file deploy-template.yaml --s3-bucket its-my-bucket-and-ill-deploy-if-i-want-to
[...snip...]
$ AWS_PROFILE=personal sam deploy --template-file deploy-template.yaml --stack-name hello-sam --capabilities CAPABILITY_IAM --region us-east-1
[...snip...]
$ AWS_PROFILE=personal aws cloudformation describe-stacks --stack-name hello-sam --query 'Stacks[].Outputs' --region us-east-1[
[...snip...]
$ curl https://[redacted].execute-api.us-east-1.amazonaws.com/Prod/hello/
{"message":"hello from us-east-1, [redacted]","visits":"1"}
$ curl https://[redacted].execute-api.us-east-1.amazonaws.com/Prod/hello/
{"message":"hello from us-east-1, [redacted]","visits":"2"}
$ curl https://[redacted].execute-api.us-east-1.amazonaws.com/Prod/hello/
{"message":"hello from us-east-1, [redacted]","visits":"3"}
```

Not bad. Not bad at all. Now let's take this show on the road!

## Going Global

Now we've got an application and it even has data that our users expect to be present. Now let's go
multi-region! There are a couple different features that will underpin our ability to do this.

First is the [API Gateway Regional Custom Domain](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-regional-api-custom-domain-create.html). 
We need to use the same custom domain name in multiple regions, the the edge-optimized custom domain won't cut it
for us since it uses CloudFront. The regional endpoint will work for us, though.

Next, we'll hook those regional endpoints up to [Route53 Latency Records](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html#routing-policy-latency)
in order to do closest-region routing and automatic failover.

Lastly, we need to way to synchronize our DynamoDB tables between our regions so we can keep those counters
up-to-date. That's where [DynamoDB Global Tables](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GlobalTables.html) 
come in to do their magic. This will keep identically-named tables in multiple regions in-sync with low latency
and high accuracy. It uses DynamoDB Streams under the hood and 'last writer wins' conflict resolution. Which probably
isn't perfect, but is good enough for most uses.

We've got a lot to get through here. I'm going to try to keep this as short and as clear as possible. If you want
to jump right to the code, you can find it in [the hello-sam-3](/2018/hello-sam-3) directory of the repo.

First things first, let's add in our regional custom domain and map it to our API. Since we're going to be using
a custom domain name, we'll need a Route53 Hosted Zone for a domain we control. I'm going to pass through the
domain name and Hosted Zone Id via a CloudFormation parameter and use it below. When you deploy you'll need
to supply your own values for these paramters.

Toss this at the top of `template.yaml` to define the parameters:

```
Parameters:
  DomainName:
    Type: String
  HostedZoneId:
    Type: String
```

Now we can create our custom domain, provision a TLS certificate for it, and configure
the base path mapping to add our API to the custom domain - put this in the `Resources` section:

```
  Certificate:
    Type: AWS::CertificateManager::Certificate
    Properties:
      DomainName: !Ref DomainName
      ValidationMethod: DNS

  CustomDomain:
    Type: "AWS::ApiGateway::DomainName"
    Properties:
      RegionalCertificateArn: !Ref Certificate
      DomainName: !Ref DomainName
      EndpointConfiguration:
        Types:
          - REGIONAL

  Mapping:
    Type: "AWS::ApiGateway::BasePathMapping"
    Properties:
      DomainName: !Ref CustomDomain
      RestApiId: !Ref ServerlessRestApi
      Stage: Prod
```

That `!Ref ServerlessRestApi` references the implicit API Gateway that is
created as part of the `AWS::Serverless::Function` Event object.

Next, we want to assign each regional custom domain to a specific Route53 record.
This will allow us to perform latency-based routing and regional failover through the use
of custom healthchecks. Let's put in a few more resources:

```
  HealthcheckFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: hello_world/
      Handler: app.health
      Runtime: nodejs8.10
      Events:
        HelloWorld:
          Type: Api
          Properties:
            Path: /health
            Method: get

  RegionalHealthcheck:
    Type: "AWS::Route53::HealthCheck"
    Properties:
      HealthCheckConfig:
        Port: "443"
        Type: "HTTPS"
        ResourcePath: "/Prod/health"
        FullyQualifiedDomainName: !Sub ${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com
        RequestInterval: "30"
        FailureThreshold: "1"
        Regions:
          - us-east-1
          - eu-west-1
          - ap-southeast-1
      HealthCheckTags:
        - Key: Name
          Value: !Sub hello-${AWS::Region}

  Record:
    Type: "AWS::Route53::RecordSet"
    Properties:
      Name: !Ref CustomDomainName
      Region: !Ref AWS::Region
      SetIdentifier: !Sub "${AWS::Region}-api"
      Type: A
      HostedZoneId: !Ref HostedZoneId
      HealthCheckId: !Ref RegionalHealthcheck
      AliasTarget:
        DNSName: !GetAtt CustomDomain.RegionalDomainName
        HostedZoneId: !GetAtt CustomDomain.RegionalHostedZoneId
```

The `AWS::Route53::Record` resource creates a DNS record and assigns it to a specific
AWS region. When your users query for your record, they will get the value for the
region closest to them. This record also has a `AWS::Route53::HealthCheck` attached to
it. This healthcheck will check your regional endpoint every 30 seconds. If your regional
endpoint has gone down, Route53 will stop considering that record when a user queries
for your domain name.

Our Route53 Healthcheck is looking at `/health` on our API, so we'd better implement
that if we want our service to stay up. Let's just drop a stub healthcheck into `app.js`.
For a real application you could perform dependency checks and stuff, but for this we'll
just return a 200:

```
exports.health = async () => {
  return {
    statusCode: 200
  };
} 
```

The last piece, we unfortunately can't control directly with CloudFormation; we'll
need to use regular AWS CLI commands. Since Global Tables span regions, it kind of makes
sense. But before we can hook up the Global Table, each individual table needs to already exist.

Through the magic of Bash scripts, we can deploy to all of our regions and create the
Global Table all in one go!

```
# For each of our 3 chosen regions
for region in 'us-east-1' 'eu-west-1' 'ap-southeast-1'; do
  # Create a deployment bucket in the region
  aws s3 mb s3://its-my-bucket-and-ill-deploy-if-i-want-to-$region
  
  # Package up and upload our artifact for the region
  sam package --template-file template.yaml \
    --s3-bucket its-my-bucket-and-ill-deploy-if-i-want-to-$region \
    --output-template-file deploy-template-$region.yaml \
    --region $region

  # Deploy!
  sam deploy --template-file deploy-template-$region.yaml \
    --stack-name hello-sam \
    --capabilities CAPABILITY_IAM \
    --parameter-overrides DomainName=$DOMAIN_NAME HostedZoneId=$HOSTED_ZONE_ID \
    --no-fail-on-empty-changeset \
    --region $region
done

# Activate our Global Table!
aws dynamodb create-global-table --global-table-name HitsTable \
  --replication-group RegionName=us-east-1 RegionName=eu-west-1 RegionName=ap-southeast-1 \
  --region us-east-1
```

For a more idempotent (but more verbose) version of this script, check out [hello-sam-3/deploy.sh](/2018/hello-sam-3/deploy.sh).

```
$ DOMAIN_NAME=[your domain name] HOSTED_ZONE_ID=[your domain's zone id] ./deploy.sh
[...deploys and sets up global table...]
```

Note: if you've never provisioned an ACM Certificate for your domain before, you may need to
check your CloudFormation output [for the validation CNAMEs](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-certificatemanager-certificate.html).

And...that's all there is to it. You have your multi-region app!

## Let's try it out

So let's test it! How about we do this:

1. Get a few hits in on our home region
1. Fail the healthcheck in our home region
1. Send a few hits to the next region Route53 chooses for us
1. Fail back to our home region
1. Make sure the counter continues at the number we expect

```
$ curl https://hello.example.com/hello
{"message":"hello from us-east-1, [redacted]","visits":"1"}
$ curl https://hello.example.com/hello
{"message":"hello from us-east-1, [redacted]","visits":"2"}
$ curl https://hello.example.com/hello
{"message":"hello from us-east-1, [redacted]","visits":"3"}
```

Cool, we've got some data. Let's failover! The easiest way to do this is just to tell Route53 that
up actually means down. Find the healthcheck id for your region using `aws route53 list-health-checks` and run:

```
$ aws route53 update-health-check --health-check-id [health check id] --invert
```

Now let's wait a minute for it to fail over and give it another shot.

```
$ curl https://hello.example.com/hello
{"message":"hello from eu-west-1, [redacted]","visits":"4"}
$ curl https://hello.example.com/hello
{"message":"hello from eu-west-1, [redacted]","visits":"5"}
$ curl https://hello.example.com/hello
{"message":"hello from eu-west-1, [redacted]","visits":"6"}
```

Look at that, another region! And it started counting at 3. That's awesome, our data was replicated. 
Okay, let's fail back, you know the drill:

```
$ aws route53 update-health-check --health-check-id [health check id] --no-invert
```

Give it a minute for the healtcheck to become healthy again and fail back. And now let's hit
the service a few more times:

```
$ curl https://hello.example.com/hello
{"message":"hello from us-east-1, [redacted]","visits":"7"}
$ curl https://hello.example.com/hello
{"message":"hello from us-east-1, [redacted]","visits":"8"}
$ curl https://hello.example.com/hello
{"message":"hello from us-east-1, [redacted]","visits":"9"}
```

Amazing. Your users will now automatically get routed to not just the nearest region, but the
nearest _healthy_ region. And all of the data is automatically replicated between all
active regions with very low latency. This grants you a huge amount of redundancy, availability,
and resilience to service, network, regional, or application failures.

Now not only can your app scale effortlessly through the use of serverless technologies, it can
failover automatically so you don't have to wake up in the middle of the night and find there's
nothing you can do because there's a network issue that is out of your control - change your
region and route around it.

## Further Reading

I don't want to take up too much more of your time, but here's some further reading if you wish
to dive deeper into serverless:

* A great Medium post by Paul Johnston on [Serverless Best Practices](https://medium.com/@PaulDJohnston/serverless-best-practices-b3c97d551535)
* SAM has [configurations for safe and reliable deployment](https://github.com/awslabs/serverless-application-model/blob/master/docs/safe_lambda_deployments.rst) and rollback using CodeDeploy!
* AWS built-in tools for serverless monitoring are lackluster at best, you may wish to look into external services like [Dashbird](https://dashbird.io) or [Thundra](https://thundra.io) once you hit production.
* [ServerlessByDesign](https://sbd.danilop.net/) is a really great web app that allows you to drag, drop, and connect various serverless components to visually design and architect your application. When you're done you can export it to a working SAM or Serverless repository!

## Biography

Norm recently joined Chewy.com as a Cloud Engineer to help them start on their Cloud transformation. 
Previously ran the Cloud Engineering team at Cimpress. 
Find him on twitter [@nromdotcom](https://twitter.com/nromdotcom).