## Automated Control Rollout in AWS Control Tower

Control Tower provides a lot of helpful features when trying to assist you to manage your multi-account environment. One of these features is the usage of the so-called **Controls**.

Control Tower Controls help you to setup guardrails making your environment more secure and helping you ensuring governance across all OUs and accounts. 

These Controls can be split up into three different groups:

- **Preventive Controls** → With the usage of Service Control Policies this type of control prevent you doing something by setting a Deny to a specific action. 
An example for a Preventive Control is the 
`Disallow Creation of Access Keys for the Root User` Control
- **Detective Controls** → This creates a Config Rule checking if the corresponding resource has a non compliant state which then can be leveraged to a remediation action. An example Control here is `Detect Whether Unrestricted Incoming TCP Traffic is Allowed`
- **Proactive Controls** → The setting of IAM Permissions is used here to proactively make sure that specific actions for specific service or user can not be done. An example is `Require AWS Lambda function policies to prohibit public access`.

## The Bad News

When using a Control Tower environment with a nested OU structure, all underlying OUs inherit the Preventive Controls set on a higher level.

Unfortunately this inheritance does not occur for Detective and Proactive Controls. As AWS continues releasing the Controls and your environment might me also expanding this can result in OUs not having the full range of neede OUs enabled. So in case you don’t want to generate an overhead when trying to activate the Controls on every underlying OU, just keep reading to see how I resolved this issue.

## Solving the Problem

The infrastructure is built as shown in the picture below. The main actions are implemented with the usage of a AWS Step Function

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2kdqncg2uv7tlyxcbtud.png)

We have two main topics to take a look here:

1. Inherit activated Controls to all underlying OUs 
2. Making sure new OUs and accounts also get the corresponding Controls

#### Let’s look at Step 1 first:

First of all is to make sure we have a place to document all the Controls we want to activate in our environment. I used the SSM Parameter store as it lets you save values in a JSON format which we will work with in this solution. This Control list is set in the Cloudformation code and then deployed to the parameter store.

As soon as a change happens to this parameter in the code a Lambda Custom Resource gets triggered. Custom Resources in Cloudformation are mostly connected with a Lambda function which gets executes as soon as the Custom Resource gets updated. So in case we update our Control List later, the Lambda function gets executed. The function the starts the execution of a Step Function which is used to enable all the Controls on all OUs in the following way.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rzhazik9rdkefq4esufr.png)

The step function first executes a parallel state:


- First State: Reading out the SSM Parameter value by using the `GetParameter` API call and transforming the JSON to an array with a separate Lambda function. This is done to let the Step Function be able to loop over the different Control values set in the Parameter Store.
- Second State: Execute a Lambda function reading out all of OUs of the organization. In case an OU should be left out, it will not be part of the output here.

Both outputs are passed to the following Map State which then first loops over all Controls which were read out earlier from the parameter store. 

An inner loop then loops over all OUs followed by a Pass State combining the Control value and the OU value. This will get passed to the `EnableControl` API call to activate the Control to the corresponding OU. Unfortunately as the automation runs in to an error in case the Control has already been activated an catch expression was inserted here to not let the Step Function fail and continue with the next OU.

The Map States then loop over every Control and over every OU making sure all combinations were handled.

#### But how do we make sure that also newly created OUs will get the Controls? 

For this case we just need to implement an EventBridge rule listening to the `ManageOrganizationalUnit` CloudTrail event, handing this over to the Step function used above. As Control Tower takes some time to also register the OU and rolling out all necessary stacks a Wait State is set in the beginning of the Step Function. This makes sure that all prior needed steps are already finished when the Control Rollout takes place. 

## How to deploy

Of course the code should be deployed on the account where Control Tower has been activated. When deploying the template you will need to insert the Controls in an ARN format, divided by commata as seen in the screenshot below. As SSM Parameter Store saves everything as a String, make sure to not forget the apostrophes

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ixj1zp3j8yc8kp8js6v5.png)

During the deployment the Custom Resource will already get triggered, so take a look at the Step Function to see how the Controls are getting enabled!
