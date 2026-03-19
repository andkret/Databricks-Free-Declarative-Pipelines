# 5.2 Connecting Databricks to an AWS Message Queue

In this bonus section, you have two options. You can follow along and implement the same pipeline or just watch me do it. But before, a word of caution:

> [!WARNING]
> The chapter's pipeline can only be built with a paid AWS account. Be also aware that AWS Kinesis costs per hour and not just by usage. So always delete the Kinesis stream before taking a break and recreate it later when needed! And lastly, if you create the paid account now, it may take up to 24 hours until all features, including Kinesis are available.

So before we built our streaming pipeline with Lakeflow Designer, we first have to do some setup steps on AWS side. Let's start with creating an AWS account. 

## Create Free AWS Account

If you have no Amazon Web Service account yet. The easiest way to find the website is through a Google search. Simply type in "AWS".

The first result should be the AWS website.

![Google search result for term 'aws'](/images/5-2/google-search.png)

Click on the link or alternatively visit this URL:

https://aws.amazon.com/

Then click on 'Create Account'.

![Create Account button on AWS website](/images/5-2/create-account-button.png)

This will lead you to the sign up page. 

![AWS sign up page](/images/5-2/sign-up-page.png)

For the root user email address, use an email address that is not assigned to an existing AWS account. Also select an account name which can later be changed.

Then click on 'Verify email address'.

![Verify email address button](/images/5-2/verify-email-address.png)

You may need to do a security check. After the check, use the verification code you've received in the email from AWS and click verify. 

![Enter the verification code received via email](/images/5-2/email-verification-code.png)

After that, you will see a success page and you need to select a password. 

> [!WARNING] 
> Choose a very secure password, especially for the root user. If someone finds your credentials and has all access rights, they can misuse your account and cause high costs on your end!

Click 'Continue'.

![Continue button on password selection](/images/5-2/continue-password-button.png)

Next select the account plan. For now, we are good with the free option. So select 'Choose free plan'.

![Account plan selection](/images/5-2/account-plan-selection.png)

Next, provide your contact information. Select 'Agree and Continue'.

![Agree and Continue button on contact information](/images/5-2/contact-infromation-continue.png)

The next step is to provide payment details. With the free account you get $200 in free credits. However, after 6 months or if you use that account a lot, you have to pay for all costs beyond the free credits. For this course, we should not reach them, but you still have to provide payment details. Select again 'Continue'.

![Continue on payment details](/images/5-2/payment-continue-button.png)

You may need to do an additional payment verification step.

Next, you need to confirm your identity. Select 'Send SMS'.

![Send SMS button](/images/5-2/send-sms.png)

Again, select 'Continue' after you enter entered the verification code.

![Enter the SMS verification code](/images/5-2/sms-verification-code.png)

Wait for the account to be created.

![Wait for the account to be finished](/images/5-2/account-creation-in-process.png)

That's it for the AWS account creation. Next we will set up the AWS CLI.

## Install & Set up AWS CLI

### Install The AWS Command Line Interface

We want to later send messages through a data stream via the command line. To do that with AWS, we have several options. But the most convenient for our case is using the AWS Command Line Interface (CLI). 

There are different installation methods depending on your operating system. Use these docs to find your OS and an appropriate installation method.

https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

I will continue doing the setup for macOS, the steps are similar for other platforms. 

Run the following command to install the AWS CLI for all users (needs `sudo` permissions) in your command line:

```bash
$ curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
$ sudo installer -pkg AWSCLIV2.pkg -target /
```

This will install the CLI for you. After successful installation, you can verify everything worked as expected by running this command:

```bash
$ aws --version
```

This should print out the current version of the installed CLI.

For me, it's `aws-cli/2.33.12 Python/3.13.11 Darwin/25.0.0 exe/arm64`.

### Setup The AWS Credentials

Next, you'll need to configure some way of authorizing yourself against AWS to show that you are later allowed to send messages through the stream via the CLI. Luckily, this is fairly easy with AWS.

There are several ways of doing that. In a production setting you almost always want to avoid long-term credentials especially while using root user that has all the permissions. 

For this tutorial, we will therefore also use AWS Management Console credentials which will handle the short-term credentials for us.

If you want to find out more what setting works best for you, check this AWS page: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html

Use the following command in your terminal:

```bash
aws login --profile databricks-course
```

This will create a new profile named `databricks-course` which should not conflict with any existing AWS credentials you've provided.

You should see a prompt the region you want to use. I did use 'eu-central-1' by typing in `eu-central-1` and hitting `Enter`. Use a region that works best for you.

Now you should see a browser window open up. There click on 'Continue with Root or IAM user'.

![Sign in options browser page](/images/5-2/sign-in-options.png)

We will use the root user, we've created before. So select 'Sign in using root user email'.

![Sign in with root user email button](/images/5-2/sign-in-root-user-button.png)

Then type in your email address and click 'Next'.

![Root user sign in page](/images/5-2/aws-root-email.png)

On the next screen enter your password and click on 'Sign in'.

![Password page](/images/5-2/password-prompt.png)

You will receive an email with a verification code. Enter the verifcation code and click on 'Verify and continue'.

This should show you a success message and you should be signed in.

![AWS CLI success page](/images/5-2/aws-cli-success.png)

Check also your terminal window where it should show a text like this:

`Updated profile databricks-course to use arn:aws:iam::[NUMBER]:root credentials.`

To additionally check the connection use the following command:

```bash
aws sts get-caller-identity --profile databricks-course
```

This should bring up connection details like this:

```json
{
    "UserId": "123456789012",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:root"
}
```

Exit the view with by typing in `q`.

## Set up Kinesis Data Stream

Now with the connection to AWS in place, we can start creating our data stream that we will use to later send events that will be handled inside Databricks.

Go to the AWS site, use the search and type in `Kinesis`. Select the first service 'Kinesis'.

![Search Kinesis and select the service](/images/5-2/aws-search-kinesis.png)

In the next screen, keep 'Kinesis Data Streams' selected and click on 'Create data stream'.

![Create a Kinesis data stream](/images/5-2/aws-create-kinesis-data-stream.png)

This will lead you to the creation screen. Select as name `sensor-readings`. We will also keep 'On-demand' as 'Capacity mode'. Lastly, keep the 'Maximum record size' as is on the lowest number possible of 1024 KiB. 

![Kinesis configuration](/images/5-2/aws-kinesis-configuration.png)

> [!WARNING]
> Again a reminder that when you create this stream, it will cost both by usage and per hour. So always turn it down if you don't work with it. For pricing details visit: https://aws.amazon.com/kinesis/data-streams/pricing/

Scroll to the bottom and hit 'Create data stream'.

![Create data stream button](/images/5-2/aws-create-data-steam-button.png)

This will lead you to the Kinesis data stream that was just created. After a few seconds, you should also see a message that is was successfully created.

![Stream creation success message](/images/5-2/aws-stream-creation-success.png)

> [!NOTE]
> To later delete your data stream. Simply navigate to its detail page and click on 'Delete'.

## Send a Test Record to Kinesis

Before we connect our new data stream with Databricks, let's first verify that we can send records to Kinesis using the CLI.

Go back to your terminal and run the following command:

```bash
aws kinesis put-record \
  --stream-name sensor-readings \
  --data "$(echo -n '{"timestamp":"2024-01-15T10:00:00","sensor_id":"sensor_01","location":"Freezer A","temperature_celsius":-18.2}' | base64)" \
  --partition-key sensor_01 \
  --profile databricks-course \
  --debug
```

This will write a record to our newly created stream with base 64 encoded data, our profile, and a partition-key.

You should see an output like this:

```json
{
    "ShardId": "shardId-000000000001",
    "SequenceNumber": "[NUMBER]"
}
```
Again exit the `less` pager via `q`.

Next, go back to AWS and select the 'Data viewer' tab of your stream.

![Data viewer tab](/images/5-2/aws-data-viewer-tab-button.png)

You will initially see nothing, but here the 'ShardId' of the response becomes relevant. You need to select a shard before finding the record.

Use the response 'ShardId' in the 'Shard' dropdown and as a 'Starting position' select `Trim Horizon`. Then select 'Get records'.

![Get records button after setting the options](/images/5-2/aws-data-viewer-get-records.png)

Now you should see our record.

> [!NOTE]
> In case your record does not show up, wait a bit but this should happen rather quickly. You can also check the logs provided via the the `--debug` flag and the 'Monitoring' tab.

You can also click on the data that was sent.

![A Kinesis record shown](/images/5-2/aws-kinesis-record.png)

This will show you either the raw data that was sent or the JSON, depending on your selection.

![Kinesis record detail page with JSON](/images/5-2/aws-kinesis-record-detail.png)

With the records sent to our Kinesis Data Stream working, we will next look into connecting our data stream to Databricks.

## Databricks to Kinesis Connection

Connecting Databricks and Kinesis is a multistep process. 

### Create an IAM Role in AWS

The first step is to create a separate role that will handle the connection between AWS and Databricks.

Back in AWS click again in the search area and search for `IAM` which stands for identity and access management. Select the first result in the services section.

![Search and select the IAM service](/images/5-2/aws-iam-search.png)

In the IAM Dashboard, use the side navigation to navigate to the 'Roles'.

![Select the roles page](/images/5-2/aws-select-roles-page.png)

On the next page select 'Create role'.

![Create a new role](/images/5-2/aws-create-role-button.png)

Then select 'Custom trust policy'.

![Select custom trust policy](/images/5-2/aws-custom-trust-policy-option.png)

Paste the following policy in the Custom trust policy editor.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": ["arn:aws:iam::414351767826:role/unity-catalog-prod-UCMasterRole-14S5ZJVKOTYTL"]
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "0000"
        }
      }
    }
  ]
}
```

This will establish a cross-account trust relationship so the role can be assumed to access services on behalf of Databricks users. The role is specified by the ARN in the principal section. It still uses a placeholder for the external ID. 

Now scroll to the bottom and select 'Next'.

![Continue after settig the placeholder](/images/5-2/aws-role-next-button.png)

Skip the permissions for now and scroll to the bottom on the 'Add permissions' screen, then again click 'Next'.

![Skip the permissions](/images/5-2/aws-skip-permissions-for-role.png)

Name the role `databricks-kinesis-role`.

![Set the role name](/images/5-2/aws-role-name.png)

At last, hit 'Create role' on the bottom of the screen.

![Finish the role creation](/images/5-2/aws-create-role-final-button.png)

You will see again a success message that the role was successfully created.

![Successful role creation](/images/5-2/aws-role-success.png)

### Create an IAM Policy in AWS

Now we need to do a second step in AWS, we need to create a policy. For that, click on 'Policies' in the side navigation.

![Policies menu item](/images/5-2/aws-policies-menu-item.png)

Next, select 'Create policy'.

![Create policy button](/images/5-2/aws-create-policy.png)

Use the JSON representation in the 'Policy editor'.

![Select the JSON editor](/images/5-2/aws-policy-json-editor.png)

The enter the following code:

```json
{
    "Version":"2012-10-17",
    "Statement": [
        {
            "Action": [                
                "kinesis:DescribeStreamSummary",
                "kinesis:ListShards",
                "kinesis:GetRecords",
                "kinesis:GetShardIterator"
            ],
            "Resource": "<AWS-KINESIS-ARN>",
            "Effect": "Allow"
        },
        {
          "Action": ["sts:AssumeRole"],
          "Resource": ["arn:aws:iam::<AWS-ACCOUNT-ID>:role/databricks-kinesis-role"],
          "Effect": "Allow"
        }
    ]
}
```

You need to replace:
- `<AWS-KINESIS-ARN>` with the ARN of your stream.
- `<AWS-ACCOUNT-ID>` with you AWS account ID.

Regarding your stream ARN, you can find it by navigating back to Kinesis and opening our 'sensor-readings' stream.

![Get your stream ARN](/images/5-2/aws-stream-arn.png)

Your AWS account ID, can be found by clicking on your user and in the menu the first entry is your account ID.

![Get your account ID](/images/5-2/aws-get-account-id.png)

This will give us everything that we need to access Kinesis from Databricks. Scroll to the bottom and click 'Next'.

![Click next to proceed](/images/5-2/aws-next-on-policy-page.png)

Now name the policy `databricks-kinesis-policy`.

![Enter the policy name](/images/5-2/aws-policy-name.png)

Scroll to the bottom and click 'Create policy'.

![Click create policy to finish](/images/5-2/aws-create-policy-final-button.png)

You should end up on with another success message for the creation of the policy.

![Success message that the policy was created](/images/5-2/aws-policy-created.png)

Let's now move quickly back to the 'Roles' in the side navigation to attach the policy.

![Select again the roles page](/images/5-2/aws-roles-after-policy.png)

Click on the previously created role.

![List of AWS Roles](/images/5-2/aws-roles-overview.png)

Then click on 'Add permissions' and 'Attach policies'.

![Detail page of our custom AWS role](/images/5-2/aws-databricks-kinesis-role-detail.png)

In the upcoming list, filter by type 'Customer managed' and select our policy.

![Add policy to the role](/images/5-2/aws-add-policy-to-role.png)

Then hit 'Add permissions'.

![Add permissions button](/images/5-2/aws-add-permissions-button.png)

This should show that our new policy was correctly attached to our role.

![Policy successfully added to role](/images/5-2/aws-policy-added-to-role.png)

### Create Service Credentials in Databricks

Let's get back to Databricks.

Use the side navigation to get to the catalogs.

![Catalog menu item](/images/5-2/databricks-catalog-menu-option.png)

Then select the cog wheel and then 'Credentials'.

![Menu option to open credentials](/images/5-2/databricks-credentials-button.png)

This will lead to the 'External Data' page. Select the 'Create credential' button.

![Create credential button](/images/5-2/databricks-create-credentials-button.png)

In the dialog, select 'Service credential', provide a name `kinesis-sensor-credential` and the ARN of the role.

![Provide the credential name](/images/5-2/databricks-credential-name.png)

You can find the ARN of the role in AWS in the IAM Roles and selecting 'databricks-kinesis-role'.

![AWS roles list](/images/5-2/aws-roles-overview.png)

On the detail page, copy the ARN of the role.

![Copy AWS role ARN](/images/5-2/aws-copy-role-arn.png)

Back in Databricks, hit 'Create',

![Create new credential via 'Create' button](/images/5-2/create-new-credential-dialog-button.png)

In the resulting screen, you will get an external ID - remember, we used a placeholder before. And also the updated trust policy that we need.

![The updated trust policy in Databricks](/images/5-2/databricks-updated-policy.png)

Use this information to update your IAM role in AWS. For that, copy the trust policy in Databricks.

Then navigate to AWS IAM and your role. Then select 'Trust relationships'.

![Trust relationships tab](/images/5-2/aws-trust-relationships-tab.png)

And now 'Edit trust policy'.

![Edit trust policy button](/images/5-2/aws-edit-trust-policy-button.png)

Now replace the current content with the copied policy from Databricks. Lastly, click 'Update policy',

![Update policy button](/images/5-2/aws-update-policy-button.png)

You should see a successful update.

![Successfully updated policy](/images/5-2/aws-successfully-updated-policy.png)

## Verify The Connection

To validate the connection, we will move back to Databricks.

Close the 'Credential created' dialog via the 'Done' button. This will lead to detail page of the credential. On that page click on 'Validate configuration'

![Validate configuration button](/images/5-2/databricks-validate-connection-button.png)

This check runs rather quickly. This should be its result:

![Successful check between Databricks and AWS Kinesis](/images/5-2/databricks-kinesis-check.png)

With the setup out of the way, we can in the next video start building our Declarative Pipeline with Lakeflow Designer that consumes streaming data.