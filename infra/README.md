# Hosting on AWS

The site is served from a private S3 bucket through CloudFront, with a free HTTPS certificate from ACM.
Everything is defined in one CloudFormation template, `site.yaml`, and GitHub Actions
(`.github/workflows/deploy.yml`) uploads the site whenever `main` changes.

```
visitor ──HTTPS──▶ CloudFront ──(signed, private)──▶ S3 bucket
                        ▲                                ▲
               ACM certificate                 GitHub Actions deploy
```

What the template creates:

| Resource | Purpose |
| --- | --- |
| S3 bucket | Holds the site files. Not public; only CloudFront can read it. Old versions are kept 30 days. |
| CloudFront distribution | Serves the site over HTTPS at `farnhamroundtable.org.uk` and `www.` |
| CloudFront function | Redirects `www.` to the bare domain and serves `/about-us/` from `/about-us/index.html` |
| ACM certificate | HTTPS certificate for both names (free) |
| Route 53 records | Point the domain at CloudFront (only if a hosted zone ID is given) |
| IAM role + GitHub OIDC provider | Lets the deploy workflow on `main` upload files, with no stored AWS keys |

## Rough cost

For a small charity site the bill should be close to nothing:

| Item | Cost |
| --- | --- |
| CloudFront | Free up to 1 TB of traffic and 10 million requests a month |
| S3 | Under 1p a month for the ~8 MB site |
| ACM certificate | Free |
| Route 53 hosted zone (optional) | $0.50 a month, plus a few cents of queries |
| Cache clears on deploy | First 1,000 paths a month free; each deploy uses one |

## DNS: the one decision to make

CloudFront needs the bare domain (`farnhamroundtable.org.uk`, no `www`) to point at it.
Most DNS providers cannot put a CNAME on a bare domain, so there are two options:

1. **Move DNS to Route 53 (recommended).** Create a hosted zone for the domain, copy every existing record
   into it first (especially the `MX` and `TXT` records for email, or email will stop working),
   then change the domain's nameservers at the registrar. Pass the zone ID as `HostedZoneId` and the
   template validates the certificate and creates the records itself.
2. **Keep DNS where it is.** Leave `HostedZoneId` empty. While the stack is being created, ACM shows
   two `CNAME` validation records (in the ACM console, or `aws acm describe-certificate`); add them at the
   DNS provider and the stack carries on. Then point `www` at the `CloudFrontDomainName` output with a
   `CNAME`, and point the bare domain at it with an `ALIAS`/`ANAME`/flattened CNAME if the provider supports one.

## Contact form

S3 and CloudFront only serve files; they cannot send email. Two ways to handle the form:

- **A form service (recommended).** Formspree, Web3Forms or similar: sign up, paste the form's URL into
  `action` on `about-us/contact/index.html`. Free tiers cover a small club's volume and include spam filtering.
- **All in AWS.** A small Lambda function behind a function URL that sends the message with SES. Keeps
  everything in one account, but SES needs the domain verified and a request to AWS to leave its "sandbox"
  before it can email anyone other than verified addresses. Not built here.

The PayPal donate buttons post straight to PayPal and work unchanged.

## First-time setup

Needs the AWS CLI signed in to the charity's AWS account with admin rights, not as the root user.
[aws-access.md](aws-access.md) walks through setting that up with IAM Identity Center. All commands use `us-east-1`,
because CloudFront only accepts certificates from that region (the site is still served worldwide).

```sh
aws cloudformation deploy \
  --region us-east-1 \
  --stack-name frt-website \
  --template-file infra/site.yaml \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides HostedZoneId=Z0123456789ABC   # omit if DNS stays elsewhere
```

If the account already has a GitHub OIDC provider, add `CreateGitHubOidcProvider=false`.
Creation takes 5 to 15 minutes, most of it the certificate and CloudFront.

Then read the outputs:

```sh
aws cloudformation describe-stacks --region us-east-1 --stack-name frt-website \
  --query "Stacks[0].Outputs" --output table
```

and add three **repository variables** in GitHub (Settings → Secrets and variables → Actions → Variables):

| Variable | Stack output |
| --- | --- |
| `AWS_DEPLOY_ROLE_ARN` | `DeployRoleArn` |
| `AWS_S3_BUCKET` | `BucketName` |
| `AWS_CLOUDFRONT_DISTRIBUTION_ID` | `DistributionId` |

These are not secrets. Finally run the **Deploy to AWS** workflow once from the Actions tab (or push to `main`).
Until the variables exist the workflow skips itself, so merging it early is harmless.

## Day to day

Merge to `main` and the site updates within a couple of minutes. To undo a deploy, revert the commit on `main`.
To tear everything down, empty the bucket and delete the stack; the bucket itself is kept by design
and must be deleted by hand.
