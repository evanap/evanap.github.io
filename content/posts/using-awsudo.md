---
title: "Using awsudo"
date: 2020-08-08T11:57:00+07:00
tags: [aws,cli]
layout: single
type: post
---

Recently I've been using this tool called `awsudo`. It's a simple tool that helps you assume roles if you use that strategy in AWS. 

## Installation

Run this command to start the installation:

```
bash <(curl https://raw.githubusercontent.com/makethunder/awsudo/master/install)
```

## Usage

If this is your first time using AWS cli, configure your credentials first:

```bash
$ aws configure
AWS Access Key ID [None]: insert-your-access-key-id
AWS Secret Access Key [None]: insert-your-secret-access-key
Default region name [None]: ap-southeast-1
Default output format [None]: json
```

Now that you have your credentials configured, you can start using `awsudo`. I use it to switch between roles I have set up in `~/.aws/config`, it looks like this:

```
[profile infra]
role_arn = arn:aws:iam::123456789012:role/infra
source_profile = default
region = ap-southeast-1
mfa_serial = arn:aws:iam::98765432100:mfa/evan
```

If you don't use MFA to assume roles you can remove the last line, but I recommend using MFA for extra layer of security. Now you can start assuming roles you have defined by running:

```bash
$ awsudo -u infra env | grep AWS
AWS_SESSION_TOKEN=AQoDYXdzEBcaoAKIYnZ67+8/BzPkkpbpR3yfv9bAQoDYXdzEBcaoAKIYnZ67+8/BzPkkpbpR3yfv9b
AWS_SECRET_ACCESS_KEY=rkCLOMJMx2DbGoGySIETU8aRFfjGxgJAzDJ6Zt+3
AWS_ACCESS_KEY_ID=AKIAIXAKX3ABKZACKEDN
AWS_DEFAULT_REGION=ap-southeast-1
```

The command outputs the credentials required to run things using AWS cli, but in at this state you would need to run it every time you want to invoke AWS cli. I made it simpler by exporting those credentials to the current bash session:

```bash
eval "$(sed 's/^/export /' <(awsudo -u infra env | grep AWS))"
```

Now you just need to run it once every time your credentials expired.