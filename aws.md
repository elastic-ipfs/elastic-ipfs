# AWS

## Configuring access 

Developing locally: [Configure your environment to communicate with AWS through access keys](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html). 

Running inside AWS services, like EC2 or lambdas: Access keys  **shouldn't be used**.  [Permissions are granted by roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2.html), which are configured in [infrastructure](https://github.com/elastic-ipfs/infrastructure).
