# Signing in to AWS without the root user

This guide sets up everyday access to the charity's AWS account so that nobody signs in as the root user
(the email address the account was created with) for normal work. Do it once, before the first-time setup in
[README.md](README.md).

## The idea in one paragraph

You never want a password or key that works forever and can do everything. So the root user is locked away with
MFA, and people sign in through **IAM Identity Center** instead. When you sign in there, you pick a *role*
in the account and AWS hands you temporary credentials that expire after a few hours. Nothing long-lived sits in
a file on your laptop, MFA is checked every time, and access can be removed in one place when someone leaves the
committee.

The older approach, an **IAM user with access keys**, is what to avoid: those keys never expire, tend to end up
in `~/.aws/credentials` or pasted somewhere, and are the most common way AWS accounts get compromised.

```
You ──password + MFA──▶ IAM Identity Center ──▶ "AdministratorAccess" role ──▶ temporary credentials (hours)
GitHub Actions ──OIDC token──▶ deploy role (made by site.yaml) ──▶ temporary credentials (1 hour)
```

## 1. Lock down the root user

Sign in at <https://console.aws.amazon.com/> as the root user one last time for setup.

1. Top-right account menu → **Security credentials**.
2. Under **Multi-factor authentication (MFA)**, choose **Assign MFA device** and add a passkey or an
   authenticator app. Adding a second device is sensible, since losing the only one locks you out of root.
3. Under **Access keys**, delete any that exist. Root should have none.
4. Make sure the root email address is one the Round Table will keep (a shared committee mailbox beats a
   personal address), and store the root password somewhere the treasurer or chair can also reach.

After this, root is only for the few jobs that need it, such as closing the account or changing its email.

## 2. Set a spending alert

Still as root (or later as your admin role): **Billing and Cost Management → Budgets → Create budget →
Use a template → Monthly cost budget**. Set it to something like $5 with your email. The site should cost
well under $1 a month, so an alert means something is wrong.

## 3. Turn on IAM Identity Center

1. Open **IAM Identity Center** from the console search bar.
2. Pick the region it should live in from the top-right menu first. London (`eu-west-2`) is a natural choice;
   it only decides where the user directory is stored, not where the website runs.
3. Choose **Enable**, and accept enabling it **with AWS Organizations**. This turns the account into the
   management account of a one-account organisation, costs nothing, and is required for signing in to the
   account through Identity Center.
4. On the **Dashboard**, note the **AWS access portal URL** (like `https://d-1234567890.awsapps.com/start`).
   Bookmark it: this is where everyone signs in from now on. You can give it a friendlier name under
   **Settings → Identity source → Customize**.
5. Under **Settings → Authentication → Multi-factor authentication**, check that users must register an MFA
   device and are prompted for it at sign-in.

## 4. Create yourself a user and give it admin rights

1. **Groups → Create group**, name it `Admins`.
2. **Users → Add user**: your username, your email, first and last name. Add it to `Admins`.
   You'll get an invitation email; accept it, set a password, and register an MFA device when asked.
3. **Permission sets → Create permission set → Predefined permission set → AdministratorAccess**.
   Set the session duration to something like 4 hours.
4. **AWS accounts** → tick the account → **Assign users or groups** → choose the `Admins` group →
   choose the `AdministratorAccess` permission set → **Submit**.

Assigning to a group rather than a person means adding another committee member later is just
"add user, put them in Admins".

Why admin rather than something narrower? Creating the website stack makes an IAM role, which the
narrower predefined sets (such as `PowerUserAccess`) can't do. If others only need to look around, give them
a second permission set with `ReadOnlyAccess` instead.

## 5. Sign in

**In the browser:** open the access portal URL, sign in with your new user and MFA, expand the account, and click
**AdministratorAccess** to reach the console. Do this instead of signing in as root.

**On the command line:** install the [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html), then run:

```sh
aws configure sso
```

It asks for:

| Prompt | Answer |
| --- | --- |
| SSO session name | `frt` |
| SSO start URL | your access portal URL |
| SSO region | the region from step 3, e.g. `eu-west-2` |
| SSO registration scopes | press Enter for the default |

A browser window opens to approve the login. Then pick the account and the `AdministratorAccess` role, and
answer:

| Prompt | Answer |
| --- | --- |
| Default client Region | `us-east-1` (where the website stack must go) |
| CLI default output format | press Enter |
| Profile name | `frt` |

Check it worked:

```sh
aws sts get-caller-identity --profile frt
```

The `Arn` should contain `AWSReservedSSO_AdministratorAccess`, not `root`. When the session expires, run
`aws sso login --profile frt` to sign in again. No keys are stored; the CLI keeps only a short-lived token
under `~/.aws/sso/cache`.

Now the commands in [README.md](README.md) work with the profile, for example:

```sh
export AWS_PROFILE=frt
aws cloudformation deploy --region us-east-1 --stack-name frt-website ...
```

## How the GitHub deploy role differs

The role above is for people. The website is deployed by a separate role, `DeployRole` in `site.yaml`,
which is for a machine and is created by the stack, so there's nothing to set up by hand:

| | Your admin role | GitHub deploy role |
| --- | --- | --- |
| Who uses it | You, after password + MFA | The deploy workflow only |
| How it proves who it is | Identity Center sign-in | A signed token GitHub issues to each workflow run (OIDC) |
| Who may use it | Members of `Admins` | Only runs from `Farnham-Round-Table/frt-website` on the `main` branch |
| What it can do | Anything in the account | Upload to the website bucket and clear the CloudFront cache, nothing else |
| Stored secrets | None | None; GitHub only holds the role's ARN, which is not a secret |

Both follow the same rule: nothing permanent to leak, and each identity can only do what its job needs.
