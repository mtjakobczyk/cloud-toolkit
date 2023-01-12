## Notes

> **Event history** allows you to view, search, and download the past 90 days of activity in your AWS account. In addition, you can create a **CloudTrail trail** to archive, analyze, and respond to changes in your AWS resources. A trail is a configuration that enables delivery of events to an Amazon S3 bucket that you specify. You can also deliver and analyze events in a trail with Amazon CloudWatch Logs and Amazon EventBridge

> CloudTrail captures actions made directly by the user or on behalf of the user by an AWS service.  You can identify if the action was taken by an AWS service with the `invokedby` field in the CloudTrail event.

> A trail can be applied to all Regions or a single Region. As a best practice, create a trail that applies to all Regions. By default, when you create a trail in the console, the trail applies to all AWS Regions

> CloudTrail supports five trails per Region. A trail that applies to all AWS Regions counts as one trail in every Region.

> Turning on a trail means that you create a trail and start delivery of CloudTrail event log files to an Amazon S3 bucket. In the CloudTrail console, logging is turned on automatically when you create a trail.

> An **organization trail** is a configuration that enables delivery of CloudTrail events in the management account, delegated administrator account, and all member accounts in an AWS Organizations organization to the same Amazon S3 bucket, CloudWatch Logs, and CloudWatch Events

> **Management events** provide information about management operations that are performed on resources in your AWS account. These are also known as control plane operations. Management events can also include non-API events that occur in your account. For example, when a user signs in to your account, CloudTrail logs the ConsoleLogin event

> **Data events** provide information about the resource operations performed on or in a resource. These are also known as data plane operations. For example: Amazon S3 object-level API activity or AWS Lambda function execution activity. Data events **are not logged by default** when you create a trail. To record CloudTrail data events, you must explicitly add to a trail the supported resources or resource types for which you want to collect activity.

> **CloudTrail Insights** events capture unusual API call rate or error rate activity in your AWS account. If you have Insights events enabled, and CloudTrail detects unusual activity, Insights events are logged to a different folder or prefix in the destination S3 bucket for your trail

> **CloudTrail Lake** lets you run fine-grained SQL-based queries on your events. You do not need to have a trail configured in your account to use CloudTrail Lake. You can save Lake queries for future use, and view results of queries for up to seven days.

> AWS Security Token Service (AWS STS) is a service that has a global endpoint and also supports region-specific endpoints. When you use an AWS STS region-specific endpoint, the trail in that region delivers only the AWS STS events that occur in that region.

> **Global service events** created by CloudFront, IAM, and AWS STS will be recorded in the region in which they were created, the US East (N. Virginia) region, `us-east-1` .

## Snippets

Lookup management events (control plane API calls) since `2021-08-01` related to the attribute `ResourceType` with value `AWS::EC2::Instance` and perform client-side filtering:
```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceType,AttributeValue=AWS::EC2::Instance \
  --start-time 2021-08-01 \
  --output=json --query $QUERY
```

Sample client-side filters:
```bash
QUERY="Events[?Resources[?ResourceName == 'i-0z13c56456456549mm3f87']].{e: EventName, res: Resources}"
```
