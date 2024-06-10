## Website deployment using OIDC

### Background

I required a strategy to automate the deployment of a website to AWS that incorporated whatever services that were necessary to achieve this.

For the moment this involves:
- Deploying to an S3 bucket
- Using Roles and Polices for separation of duties
- OpenID Connect to provide identity services between AWS and Github
- Github for CI/CD as the website grows/evolves

### Documentation

- [Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- [Configure AWS Credentials for GitHub Actions](https://github.com/aws-actions/configure-aws-credentials#configure-aws-credentials-for-github-actions)
- [Configuring a role for GitHub OIDC identity provider](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html#idp_oidc_Create_GitHub)

### Configuration

#### Create the S3 Bucket

The first step is to create the S3 bucket, either through the web portal or via the terminal as this will be required to configure the policies and the Github action.

```bash
$ aws s3api create-bucket --bucket $bucket_name --region $region
```

#### Adding the secrets to Github

Three secrets will need to be defined: 
- AWS_REGION,
- AWS_ROLE_TO_ASSUME,
- AWS_S3_BUCKET_NAME.

AWS_REGION is required for the AWS actions, as you will be logging into a specific region to perform these actions, even though some occur in ue-east-1 regardless.

AWS_ROLE_TO_ASSUME is the arn of the role that is defined in the next step.

AWS_S3_BUCKET_NAME is the bucket name from the previous.

#### Bucket policy for role

The least amount of privileges required for the role:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObjectAcl",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:DeleteObject",
                "s3:PutObjectAcl"
            ],
            "Resource": [
                "arn:aws:s3:::s3-bucket-name*",
                "arn:aws:s3:::s3-bucket-name*/*"
            ]
        }
    ]
}
```

This means that the role can only create and delete objects, create and delete a directory structure, list the buckets contents, and access basic ACL properties of the specified bucket(s).

#### Trust relationship

The trust relationship defines what repository is authorised to act within the defined role.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::467812895274:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:repo-owner/repo:*",
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
```

It should be noted that ```"repo:repo-owner/repo:*"``` will give an warning as the visual editor indicates that wildcard values can be a issue as it can allow requests from unintended sources to also act in the role.

When setting the the ```secrets.AWS_ROLE_TO_ASSUME``` it the full ARN ```arn:aws:iam::...:role/role_name_goes_here``` and not the friendly name.