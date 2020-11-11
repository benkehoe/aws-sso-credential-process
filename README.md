# aws-sso-credential-process
**Get AWS SSO working with all the SDKs that don't understand it yet**

> :warning: This tool has been folded into [aws-sso-util](https://github.com/benkehoe/aws-sso-util), which includes more functionality. You can continue to use this package, but I won't be adding new features here.

Currently, [AWS SSO](https://aws.amazon.com/single-sign-on/) support is implemented in the [AWS CLI v2](https://aws.amazon.com/blogs/developer/aws-cli-v2-is-now-generally-available/), but the capability to usage the credentials retrieved from AWS SSO by the CLI v2 has not been implemented in the various AWS SDKs. However, they all support the [credential process](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sourcing-external.html) system. This tool bridges the gap by implementing a credential process provider that understands the SSO credential retrieval and caching system. Once AWS implements the necessary support in the SDK for your favorite language, this tool will no longer be necessary.

If you try this and your tools still don't work with the credentials, you can get the credentials themselves using [aws-export-credentials](https://github.com/benkehoe/aws-export-credentials), which can also inject them as environment variables for your program.

## SDK support for AWS SSO

Read this section to determine if the SDK in your language of choice has implemented support for AWS SSO.

* [boto3 (the Python SDK)](boto3.amazonaws.com/v1/documentation/api/latest/index.html) has added support for loading credentials cached by [`aws sso login`](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/sso/login.html) as of [version 1.14.0](https://github.com/boto/boto3/blob/develop/CHANGELOG.rst#1140). However, it does not support initiating authentication. That is, if the credentials are expired, you have to use `aws sso login` to login again, and this of course means that you (and your users) need the AWS CLI v2 installed for your Python scripts to use AWS SSO credentials. `aws-sso-credential-process` does not have a dependency on AWS CLI v2 and supports initiating authentication.

## Quickstart

1. You can install with `pip` on any platform, but I recommend you use [`pipx`](https://pipxproject.github.io/pipx/), which will let you install `aws-sso-credential-process` in an isolated virtualenv while linking the executables onto your `$PATH`. To install `pipx`:

Mac:
```bash
brew install pipx
pipx ensurepath
```

Other:
```bash
python3 -m pip install --user pipx
python3 -m pipx ensurepath
```

2. Install the tool.
```bash
pipx install aws-sso-credential-process
```

3. Set up your `.aws/config` file for AWS SSO as normal (see step 6 for how to make this easier):

```
[profile my-sso-profile]

region = us-east-2
output = yaml

sso_start_url = https://something.awsapps.com/start
sso_region = us-east-2
sso_account_id = 123456789012
sso_role_name = MyLeastPrivilegeRole
```

4. Then, just add a [`credential_process` entry](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sourcing-external.html) to the profile, using the `--profile` flag with the same profile name (see step 6 for how to make this easier):

```
[profile my-sso-profile]

credential_process = aws-sso-credential-process --profile my-sso-profile

region = us-east-2
output = yaml

sso_start_url = https://something.awsapps.com/start
sso_region = us-east-2
sso_account_id = 123456789012
sso_role_name = MyLeastPrivilegeRole

```

5. You're done! Test it out:
```bash
aws sso login --profile my-sso-profile
python -c "import boto3; print(boto3.Session(profile_name='my-sso-profile').client('sts').get_caller_identity())"
```

NOTE: if you test it out with your favorite script or application and get something like `NoCredentialProviders: no valid providers in chain.`, you may need to set the environment variable `AWS_SDK_LOAD_CONFIG=1`. The Go SDK, and applications built with the Go SDK (like Terraform) [don't automatically use your `.aws/config` file](https://docs.aws.amazon.com/sdk-for-go/v1/developer-guide/configuring-sdk.html).


6. Streamline the process. If you've got one main AWS SSO configuration, set up your `.bashrc` (or similar) like this:
```
export AWS_CONFIGURE_SSO_DEFAULT_SSO_START_URL=https://something.awsapps.com/start
export AWS_CONFIGURE_SSO_DEFAULT_SSO_REGION=us-east-2
```

Use `aws-configure-sso-profile` to set up your AWS SSO profiles. This will set up your profile as shown above interactively, including prompting you to select from available accounts and roles. It will look something like this:
```
$ aws-configure-sso-profile --profile my-sso-profile
SSO start URL [https://something.awsapps.com/start]:
SSO Region [us-east-2]:
Attempting to automatically open the SSO authorization page in your default browser.
If the browser does not open or you wish to use a different device to authorize this request, open the following URL:

https://device.sso.us-east-2.amazonaws.com/

Then enter the code:

ABCD-WXYZ
There are N AWS accounts available to you.
Using the account ID 123456789012
The only role available to you is: MyLeastPrivilegeRole
Using the role name "MyLeastPrivilegeRole"
CLI default client Region [None]: us-east-2
CLI default output format [None]: yaml

To use this profile, specify the profile name using --profile, as shown:

aws s3 ls --profile my-sso-profile
```

## Configuration

The `aws-configure-sso-profile` tool wraps `aws configure sso` to help you set up profiles in `.aws/config`; you can set the environment variables `AWS_CONFIGURE_SSO_DEFAULT_SSO_START_URL` and `AWS_CONFIGURE_SSO_DEFAULT_SSO_REGION` to set defaults for those values so you're not typing them all the time. The tool will set up the `credential_process` entry as well. Note that `--profile` is required (unlike `aws configure sso`).

The order of configuration matches the AWS CLI and SDKs: values from CLI parameters take precedence, followed by env vars, followed by settings in `.aws/config`.

The `--profile` parameter on `aws-sso-credential-process` doesn't work like the same parameter on the AWS CLI, and cannot be set from the environment; it's intended only to help make the `credential_process` entry in a profile more concise.

### Interactive authentication

The most important thing to determine is whether or not you want to allow interactive authentication, which is off by default (so that the behavior is the same as the AWS CLI v2).

When interactive authentication is off, you need to use the CLI v2's `aws sso login` to login through AWS SSO. If you haven't logged in or your session has expired, the process will fail and interrupt whatever you're doing.

With interactive authentication turned on, the same functionality of `aws sso login` will be triggered automatically; a browser will pop up to prompt you to log in (or, if you're already logged in, it will prompt you to approve the login). This is useful when you're running scripts interactively, but bad for automated processes that are incapable of logging in.

Note that with interactive authentication off, you have to have the AWS CLI v2 installed, but with interactive authentication on, this dependency is eliminated.

**To enable interactive authentication, the best way is to set `AWS_SSO_INTERACTIVE_AUTH=true` in your environment.** This lets you control whether interactive auth is enabled for a given profile depending on the situation you're using it for. Otherwise, you can set `sso_interactive_auth=true` in your profile in `.aws/config`, or use the `--interactive` flag for the process. Note that you can use the `--noninteractive` flag to disable interactive auth even if the environment variable is set.

When setting up a profile using `aws-configure-sso-profile`, you can use `--set-auth-interactive` or `--set-auth-noninteractive` to fix that profile as either interactive or noninteractive, respectively.

Note that if you've got your profile set up as shown above, the AWS CLI v2 won't get interactive authentication, because it will natively use the profile configuration, skipping this tool as a credential process. If you *really* want interactive auth with the CLI, you could put the AWS SSO configuration information as parameters to the tool in the credential process directive, instead of directly in the profile, and then the CLI will use credential process as well, but I don't really recommend this route.

### Debugging
Setting the `--debug` flag or the env var `AWS_SSO_CREDENTIAL_PROCESS_DEBUG=true` will cause debug output to be sent to `.aws/sso/aws-sso-credential-process-log.txt`. Note that this file will contain your credentials, though these credentials are both short-lived and also cached within the same directory.

### Account

* `.aws/config`: `sso_account_id`
* env var: `AWS_SSO_ACCOUNT_ID`
* parameter: `--account-id`

### Role

* `.aws/config`: `sso_role_name`
* env var: `AWS_SSO_ROLE_NAME`
* parameter: `--role-name`

### SSO Start URL

* `.aws/config`: `sso_start_url`
* env var: `AWS_SSO_START_URL`
* parameter: `--start-url`

### SSO Region

* `.aws/config`: `sso_region`
* env var: `AWS_SSO_REGION`
* parameter: `--region`
