# Automate VPC Prefix List & WAF IPSets with AWS IP Ranges

This project creates a Lambda function that automatically creates or updates AWS VPC Prefixes & WAF IPSets with AWS service's IP ranges from the [ip-ranges.json](https://docs.aws.amazon.com/general/latest/gr/aws-ip-ranges.html) file.

You can configure which services and region ranges to create VPC Prefixes & WAF IPSets for as well as enable Organization wide sharing using Resource Access Manager (RAM).

Use cases include allowing CloudFront requests, API Gateway requests, Route53 health checker and EC2 IP range (which includes AWS Lambda and CloudWatch Synthetics).  
The resources are created or updated in the region where the CloudFormation stack is created.

Important: this application uses various AWS services and there are costs associated with these services after the Free Tier usage - please see the [AWS Pricing page](https://aws.amazon.com/pricing/) for details. You are responsible for any AWS costs incurred. No warranty is implied in this example.

> **NOTE**  
> This is an upgraded version of the repository below and integrates Lambda PowerTools. This repo adds support for RAM Prefix sharing and also moves the service configuration into AWS AppConfig.
> https://github.com/aws-samples/update-aws-ip-ranges

## Requirements

* [Create an AWS account](https://portal.aws.amazon.com/gp/aws/developer/registration/index.html) if you do not already have one and log in. The IAM user that you use must have sufficient permissions to make necessary AWS service calls and manage AWS resources.
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) installed and configured
* [Git Installed](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
* [AWS Serverless Application Model](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html) (AWS SAM) installed

## Deployment Instructions

1. Create a new directory, navigate to that directory in a terminal and clone the GitHub repository:

    ```shell
    git clone https://github.com/sjramblings/update-aws-ip-ranges
    ```

1. Change directory to the pattern directory:

    ```shell
    cd update-aws-ip-ranges
    ```

1. From the command line, use AWS SAM to deploy the AWS resources for the pattern as specified in the template.yml file:

    ```shell
    sam deploy --guided
    ```

1. During the prompts:
    * Enter a stack name
    * Enter the desired AWS Region
    * Allow SAM CLI to create IAM roles with the required permissions.

    Once you have run `sam deploy --guided` mode once and saved arguments to a configuration file (samconfig.toml), you can use `sam deploy` in future to use these defaults.

1. Note the outputs from the SAM deployment process. These contain the resource names and/or ARNs which are used for testing.

## Testing

To test the lambda send test data using the following command. Be sure to update the Function Name from the Cloudformation Stack Output.

```shell
aws lambda invoke \
  --function-name FUNCTION_NAME \
  --cli-binary-format 'raw-in-base64-out' \
  --payload file://events/event.json lambda_return.json
```

After successful invocation, you should receive the response below with no errors.

```json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

## Configuration

To configure which service the lambda should handle IP ranges or which region, you need to change the file AppConfig hosted configuration.  

To see the list of possible service names inside `ip-ranges.json` file, run the command below:

```shell
curl -s 'https://ip-ranges.amazonaws.com/ip-ranges.json' | jq -r '.prefixes[] | .service' | sort -u
```

To see the list of possible region names inside `ip-ranges.json` file, run the command below:

```shell
curl -s 'https://ip-ranges.amazonaws.com/ip-ranges.json' | jq -r '.prefixes[] | .region' | sort -u
```

See below the file commented.

```shell
{
    "Services": [
        {
            # Service name. MUST match the service name inside ip-ranges.json file.
            # Case is sensitive.
            "Name": "API_GATEWAY",
            
            # Region name. It is an array, so you can specify more than one region. MUST match the region name inside ip-ranges.json file.
            # Case is sensitive.
            #
            # Please not that there is one region called GLOBAL inside ip-ranges.json file.
            # If you want to get IP ranges from all region keep the array empty.
            #
            # If you specify more than one region, or keep it empty, it will aggregate the IP ranges from those regions inside the resource at the region where Lambda function is running.
            # It will NOT create the resources on each region specified.
            "Regions": ["sa-east-1"],
            
            "PrefixList": {
                # Indicate if VPC Prefix List should be create for IP ranges from this service. It will be created in the same region where Lambda function is running.
                "Enable": true,
                # Indicate if VPC Prefix List IP ranges should be summarized or not for this specific service.
                "Summarize": true
            },
            
            "WafIPSet": {
                # Indicate if WAF IPSet should be create for IP ranges from this service. It will be created in the same region where Lambda function is running.
                "Enable": true,
                # Indicate if WAF IPSet IP ranges should be summarized or not for this specific service.
                "Summarize": true,
                # WAF IPSet scope to create or update resources. Possible values are ONLY "CLOUDFRONT" and "REGIONAL".
                # Case is sensitive.
                #
                # Note that "CLOUDFRONT" can ONLY be used in North Virginia (us-east-1) region. So, you MUST deploy it on North Virginia (us-east-1) region.
                "Scopes": ["CLOUDFRONT", "REGIONAL"]
            }
        }
    ]
}
```

Example:

```json
{
    "Services": [
        {
            "Name": "API_GATEWAY",
            "Regions": ["sa-east-1"],
            "PrefixList": {
                "Enable": true,
                "Summarize": true
            },
            "WafIPSet": {
                "Enable": true,
                "Summarize": true,
                "Scopes": ["REGIONAL"]
            }
        },
        {
            "Name": "CLOUDFRONT_ORIGIN_FACING",
            "Regions": [],
            "PrefixList": {
                "Enable": false,
                "Summarize": false
            },
            "WafIPSet": {
                "Enable": true,
                "Summarize": false,
                "Scopes": ["REGIONAL"]
            }
        },
        {
            "Name": "EC2_INSTANCE_CONNECT",
            "Regions": ["sa-east-1"],
            "PrefixList": {
                "Enable": true,
                "Summarize": false
            },
            "WafIPSet": {
                "Enable": true,
                "Summarize": false,
                "Scopes": ["REGIONAL"]
            }
        },
        {
            "Name": "ROUTE53_HEALTHCHECKS",
            "Regions": [],
            "PrefixList": {
                "Enable": true,
                "Summarize": false
            },
            "WafIPSet": {
                "Enable": true,
                "Summarize": false,
                "Scopes": ["REGIONAL"]
            }
        }
    ]
}
```

## Notes

* WAF IPSet will just be updated if there are entries to remove or to add.
* VPC Prefix List will just be updated if there are entries to remove or to add.
* When VPC Prefix List is created, the `max entries` configuration will be the length of current IP ranges for that service plus 10.
* When VPC Prefix List is updated, if current `max entries` configuration is lower than the length of current IP ranges for that service, it will change the `max entries` to the length of current IP ranges. If it fails to update, due to size restriction where Prefix List is used, it will NOT update the IP ranges.
* If it fails to create or update resource for any service, the code will not stop, it will continue to handle the other resource and services.
* It only creates resource for service and IP version if there is at least one IP range. Otherwise, it will not create.
* Resources are named as `aws-ip-ranges-<SERVICE_NAME>-<IP_VERSION>`.  
Where:  
  * `<SERVICE_NAME>` is the service name inside `ip-ranges.json` file. Converted to lower case and replaced `_` with `-`.  
  * `<IP_VERSION>` is `ipv4` or `ipv6`.

Examples:
* `aws-ip-ranges-api-gateway-ipv4`
* `aws-ip-ranges-route53-healthchecks-ipv4`
* `aws-ip-ranges-route53-healthchecks-ipv6`


## Cleanup
 
1. Delete the stack
    ```bash
    aws cloudformation delete-stack --stack-name [YOUR STACK NAME]
    ```
1. Confirm the stack has been deleted
    ```bash
    aws cloudformation list-stacks --query "StackSummaries[?contains(StackName,'[YOUR STACK NAME]')].StackStatus"
    ```
----

### 2. Reference resources

For WAF IPSet, see [Using an IP set in a rule group or Web ACL](https://docs.aws.amazon.com/waf/latest/developerguide/waf-ip-set-using.html).  
For VPC Prefix List, see [Reference prefix lists in your AWS resources](https://docs.aws.amazon.com/vpc/latest/userguide/managed-prefix-lists-referencing.html).
For Resource Access Manager, see [Enable resource sharing within AWS Organizations](https://docs.aws.amazon.com/ram/latest/userguide/getting-started-sharing.html#getting-started-sharing-orgs)


## Troubleshooting

**Wrong WAF IPSet Scope**

> An error occurred (WAFInvalidParameterException) when calling the ListIPSets operation: Error reason: The scope is not valid., field: SCOPE_VALUE, parameter: CLOUDFRONT

Scope name `CLOUDFRONT` is correct, but it MUST be running on North Virginia (us-east-1) region. If it runs outside North Virginia, you will see the error above.  
Please make sure it is running on North Virginia (us-east-1) region.


SPDX-License-Identifier: MIT-0
