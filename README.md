# Automating ElasticSearch with CloudFormation

Getting to provision ElasticSearch clusters with CloudFormation can be a daunting task. You will look at the CloudFormation User Guide, and see that [Elasticsearch](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-elasticsearch-domain.html) is supported, and start merrily on your way to automation heaven.

My experience getting everything to work as desired has been more of a hassle than I expected, so I'm going to document what I found along the way:

## ElasticSearch VPC needs a ServiceRole to provision correctly

When you create an Elasticsearch cluster inside a VPC, you have to create an [IAM Service Linked Role](https://docs.aws.amazon.com/IAM/latest/UserGuide/using-service-linked-roles.html) so that 
the Elasticsearch service can see what your VPC looks like, create Network Interfaces, etc.

When you create your first Elasticsearch Cluster from the console, AWS does this for you. If you are using an account where you have never created an Elasticsearch cluster from
the console, CloudFormation will fail to provision the cluster with the following error:

```
Before you can proceed, you must enable a service-linked role to give Amazon ES permissions to access your VPC. 
(Service: AWSElasticsearch; Status Code: 400; Error Code: ValidationException; Request ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx; Proxy: null)
```

There is a [CloudFormation Resource for ServiceLinkedRoles](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-servicelinkedrole.html). You happily put that 
in your template, and it works. Great! (for a while)

```
Resources:
  BasicSLR:
    Type: 'AWS::IAM::ServiceLinkedRole'
    Properties:
      AWSServiceName: es.amazonaws.com
```

If you call create-stack again, to instance the template again, you get an error because the Elasticsearch ServiceLinkedRole already exists in IAM!

Worse even: if you do this in an account which already had an Elasticsearch Cluster provisioned, it will also fail (because the role was already created).

Since you read the whole documentation for the resource, you remember the [CustomSuffix](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-servicelinkedrole.html#cfn-iam-servicelinkedrole-customsuffix) property of the ServiceLinkedRole, which says:


> A string that you provide, which is combined with the service-provided prefix to form the complete role name. 
If you make multiple requests for the same service, then you must supply a different CustomSuffix for each request

Yay! They have thought of this for me! I can put a `!Ref AWS::StackName` in the suffix!

The next error in CloudFormation is desolating:

```
Custom suffix is not allowed for es.amazonaws.com
```

## Writing a Custom Resource to automate ServiceLinkedRoles

You always have the option to resort to CloudFormation [Custom Resources](https://docs.aws.amazon.com/en_us/AWSCloudFormation/latest/UserGuide/template-custom-resources-lambda.html) and Lambda functions to get CloudFormations' small gaps filled in, but I really dread doing this:

1. It always introduces the overhead of testing and getting things right. Even if you find the code for the Lambda Function on some blog post, you have to modify it.
   I've never found a Custom Resource Lambda Function online that completely works for me. They normally are oblivious of updates, don't handle exceptions, don't always return responses
   (so your stacks get "stuck" for hours). Sometimes they swallow an incoming update, reporting success, but not doing any action underneath. Don't get me wrong, I thank the people that 
   share their experience and code! I write this article as a way of giving back, but you have to know that when you "adopt" someone elses Custom Resource Lambda (even mine), you very 
   probably have to tailor it to work for your use cases.
1. You always have the impression that you're doing lots of work that AWS should have already done (I think AWS CloudFormation should be a first-class citizen) and you always have the
   impression that you're working hard, when "some time later" (days? weeks?) AWS will just render your effort obsolete.

Since I needed stuff to work now, and I thought that a resource that ***ensures*** that a Service Linked Role is present, instead of creating it, would be easy enough to build, I
went ahead. Updates would be handled by creating a new role, and deletes would be handled by not doing anything (AWS Service Linked Roles are left around all the time, since lots of services
automatically create the Role on first use, and they never get deleted).

Here is [my take](servicelinkedrole.yaml) of a Cloudformation Custom Resource for Service Linked Roles that don't support Custom Prefixes.

You can inline the resources from the template in your stack, or provision the template as a stack, and then use the Lambdas' ARN. You choose.

You use the resource in your template like this:
```
  ESServiceRole:
    Type: Custom::EnsuredServiceRole
    Version: '1.0'
    Properties:
      ServiceToken: !GetAtt EnsuredServiceRole.Arn
      Service: es.amazonaws.com
```

So now my trip was seemingly going to be pleasant (or so I thought).

## Enabling ElasticSearch to log to CloudWatch Logs

The Elasticsearch CloudFormation resource lets you configure logs coming from the ElasticSearch server to CloudWatch Logs.

You innocently create some log groups in the template for each type of log you want configured:

```
Resources:
  ElasticSearch:
    Type: AWS::Elasticsearch::Domain
    Properties:
      [...]
      LogPublishingOptions:
        SEARCH_SLOW_LOGS:
          CloudWatchLogsLogGroupArn: !GetAtt LogGroupSearchSlow.Arn
          Enabled: true
        INDEX_SLOW_LOGS:
          CloudWatchLogsLogGroupArn: !GetAtt LogGroupIndexSlow.Arn
          Enabled: true
        ES_APPLICATION_LOGS:
          CloudWatchLogsLogGroupArn: !GetAtt LogGroupApplication.Arn
          Enabled: true
      [...]
  LogGroupSearchSlow:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Join [ '', [ '/', { Ref: 'AWS::StackName' }, '/es/search-slow' ] ] 
      RetentionInDays: 7
  LogGroupIndexSlow:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Join [ '', [ '/', { Ref: 'AWS::StackName' }, '/es/index-slow' ] ] 
      RetentionInDays: 7
  LogGroupApplication:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Join [ '', [ '/', { Ref: 'AWS::StackName' }, '/es/application' ] ] 
      RetentionInDays: 7
```

What is your surprise when you get an error in CloudFormation:
```
The Resource Access Policy specified for the CloudWatch Logs log group /xxxxxxxxx/es/search-slow does not grant 
sufficient permissions for Amazon Elasticsearch Service to create a log stream. Please check the Resource Access Policy. 
(Service: AWSElasticsearch; Status Code: 400; Error Code: ValidationException; Request ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx; Proxy: null)
```

This is not expected, because you can't find any reference or documentation in the CloudFormation User Guide advising you of such thing.

I even got confused, and tried to attach a AWS::IAM::Policy resource to the ElasticSearch Service role so that the ElasticSearch service could
write to the Log groups. This failed pretty badly, as you cannot modify Service Roles.

I finally [found a poor soul in my same situation](https://stackoverflow.com/questions/62912027/cloudwatch-resource-access-policy-error-while-creating-amazon-elasticsearch-serv)

It results that you have to tell CloudWatch logs to let the ElasticSearch service write to the LogGroups. [There is no resource in CloudFormation to do this](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/AWS_Logs.html). This is done by calling IAM `PutResourcePolicy` on the Logs service:

```
SERVICE=es.amazonaws.com
ARN=arn:aws:logs:us-east-1:xxxxxxxxxxxx:log-group:/xxxxxxxxx/es/application:*
aws logs put-resource-policy --policy-name POLICY_NAME --policy-document "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"$SERVICE\"},\"Action\":[\"logs:CreateLogStream\",\"logs:PutLogEvents\"],\"Resource\":\"$ARN\"}]}"
```

There's no way of doing this from CloudFormation :(

## Writing a Custom Resource to automate PutResourcePolicy

I've already stated how unconfortable it feels to be writing a CloudFormation Custom Resource...

But since I need this stuff to work now, and I found someone with a similar problem, and some [code](https://gist.github.com/sudharsans/cf9c52d7c78a81818a4a47872982bd76]), 
I went ahead and solved this issue too.

You can find the code for my take on the CloudWatch Logs PutResourcePolicy Resource [here](permission_customresource.yaml). Note that the `service` is hardcoded to `es.amazonaws.com`. Changing it, or making it a parameter in the stack is left as ani exercise to the reader (PRs welcome).

You can inline the resources from the template in your stack, or provision the template as a stack, and then use the Lambdas' ARN. Your choice.

To use the Custom Resource:
```
  ResourcePolicyForSearchSlow:
    Type: Custom::AddResourcePolicy
    Version: '1.0'
    Properties:
      ServiceToken: !GetAtt GiveLogsWritePermissionTo.Arn
      LogGroupArn: !GetAtt LogGroupSearchSlow.Arn
  ResourcePolicyForIndexSlow:
    Type: Custom::AddResourcePolicy
    Version: '1.0'
    Properties:
      ServiceToken: !GetAtt GiveLogsWritePermissionTo.Arn
      LogGroupArn: !GetAtt LogGroupIndexSlow.Arn
  ResourcePolicyForApplication:
    Type: Custom::AddResourcePolicy
    Version: '1.0'
    Properties:
      ServiceToken: !GetAtt GiveLogsWritePermissionTo.Arn
      LogGroupArn: !GetAtt LogGroupApplication.Arn
```

Note that on your ElasticSearch cluster, you must `DependOn` these resources, to assure they are created before CloudFormation creates the Elasticsearch cluster:

```
Resources:
  ElasticSearch:
    Type: AWS::Elasticsearch::Domain
    DependsOn:
    - AddResourcePolicyForIndexSlow
    - AddResourcePolicyForIndexSlow
    - AddResourcePolicyForApplication
    Properties:
      [...]
```

# Conclusions

I feel AWS could do a lot to improve the user experience for automating ElasticSearch clusters, as there are some sharp edges:
 - AWS could put a new property on the `AWS::IAM::ServiceLinkedRole` object that assures that the role exists (and doesn't delete it on stack delete).
 - AWS should document in the CloudFormation User Guide that you have to call PutResourcePolicy on CloudWatch logs for ElasticSearch to be able to write to the appropiate Log Groups (note that Route 53 has a configuration with the same problem). This would save a lot of misunderstanding, second guessing, etc.
 - AWS should really write a CloudWatch Logs resource for calling PutResourcePolicy on CloudWatch Logs. This is kind-of a big oversight, since there are people struggling with this. You have two pieces available in CloudFormation, but you don't have the glue to stick them together.

# Author, Copyright and License

This article was authored by Jose Luis Martinez Torres.

This article is (c) 2020 Jose Luis Martinez Torres, Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).

The canonical, up-to-date source is [GitHub](https://github.com/pplu/cfn-elasticsearch). Feel free to contribute back.
