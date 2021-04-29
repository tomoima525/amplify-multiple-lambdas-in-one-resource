This repo showcases adding multiple lambda functions in one resource using Amplify

# Why do you need this?

When adding a new function with Amplify, it will create a new stack on CloudFormation. If your project or team become larger this could cause an issue because CloudFormation allows you to create [200 stacks maximum per AWS account](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html).

As a workaround, we can deploy multiple functions from one resource. Another benefit of this approach is that we can share the same policies/executionRole if each function are in the same domain.

# Output of this repository

You can find two functions created in one resource.

![Screen Shot 2021-04-29 at 11 50 23 AM](https://user-images.githubusercontent.com/6277118/116606477-a25e5380-a8e5-11eb-8eae-5712f32a68bc.png)

![Screen Shot 2021-04-29 at 11 54 42 AM](https://user-images.githubusercontent.com/6277118/116606519-b43ff680-a8e5-11eb-9d81-ac282f5df68e.png)

# Downside of this approach

- All codes under the same resource will be uploaded to each lambda function.

# What needs to be added

Let's say you want to add a new function `LambdaFunctionTwo`. You need to add the below setup under `Resource` property.

```
"Resource": {
    ...,
    "LambdaFunctionTwo": {
      "Type": "AWS::Lambda::Function",
      "Metadata": {
        "aws:asset:path": "./src",
        "aws:asset:property": "Code"
      },
      "Properties": {
        "Code": {
          "S3Bucket": {
            "Ref": "deploymentBucketName"
          },
          "S3Key": {
            "Ref": "s3Key"
          }
        },
        "Handler": "functionTwo/index.handler",  // You must set the path to the actual function
        "FunctionName": {
          "Fn::If": [
            "ShouldNotCreateEnvResources",
            "multiplefunctionec0b1d53",
            {
              "Fn::Join": [
                "",
                [
                  "multiplefunctionec0b1d53",
                  "-",
                  "functionTwo-", // function name should be unique
                  {
                    "Ref": "env"
                  }
                ]
              ]
            }
          ]
        },
        "Environment": {
          "Variables": {
            "ENV": {
              "Ref": "env"
            },
            "REGION": {
              "Ref": "AWS::Region"
            }
          }
        },
        "Role": {
          "Fn::GetAtt": [
            "LambdaExecutionRole",
            "Arn"
          ]
        },
        "Runtime": "nodejs14.x",
        "Layers": [],
        "Timeout": "25"
      }
    }
}
```

Then you need to add logging permissions under `lambdaexecutionpolicy`

```
"lambdaexecutionpolicy": {
  "PolicyDocument": {
    "Resource": [
      {...},
      {
        "Fn::Sub": [
          "arn:aws:logs:${region}:${account}:log-group:/aws/lambda/${lambda}:log-stream:*",
          {
            "region": {
              "Ref": "AWS::Region"
            },
            "account": {
              "Ref": "AWS::AccountId"
            },
            "lambda": {
              "Ref": "LambdaFunctionTwo"  // reference to LambdaFunctionTwo
            }
          }
        ]
      }
    ]
```

Lastly if you want to expose the out put add this to `Outputs` section

```
"Outputs": {
  ...,
  "FunctionTwoName": {
    "Value": {
      "Ref": "LambdaFunctionTwo"
    }
  },
  "FunctionTwoArn": {
    "Value": {
      "Fn::GetAtt": [
        "LambdaFunctionTwo",
        "Arn"
      ]
    }
  },
}
```

# Gotchas

- When you are adding a new function in an existing resource, you can not change the LogicalID of an existing resource. (e.g. `LambdaFunction`) If you change `amplify push` will fail with a message `Cannot read property 'Type' of undefined`
- When you update cloud-formation-template.yml manually you need to call `amplify env checkout {your env}` so that `amplify/backend/amplify-meta.json`(hidden file) gets updated locally.
- When removing a lambda function, you just need to remove the resource from cloud-formation-template.yml
