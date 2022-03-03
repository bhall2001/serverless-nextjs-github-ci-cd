# serverless-nextjs-github-ci-cd

This is an opinionated view of how to setup a [Next.js](https://nextjs.org/) project bootstrapped with [`create-next-app`](https://github.com/vercel/next.js/tree/canary/packages/create-next-app) using [Serverless Nextjs Component](https://github.com/serverless-nextjs/serverless-next.js) with GitHub Actions for ci/cd.

## Motivation

There is a desire to implement ci/cd process with the Serverless Nextjs Component with environments where checking into a branch in a GitHub repository performs ci/cd tasks resulting in a staging environment while tagging a commit with a version number (or creating a release in GitHub) performs ci/cd tasks for a production environment.

Sites from ci/cd process:

[Resulting staging environment](https://staging-your-site-name.bobhall.net)

[Resulting production environment](https://prod-your-site-name.bobhall.net)

## Contents

- [serverless-nextjs-github-ci-cd](#serverless-nextjs-github-ci-cd)
  - [Motivation](#motivation)
  - [Contents](#contents)
  - [Overview](#overview)
  - [Getting started](#getting-started)
  - [Install serverless](#install-serverless)
  - [Create S3 serverless assets storage bucket](#create-s3-serverless-assets-storage-bucket)
  - [Setup GitHub secrets](#setup-github-secrets)
  - [High level environment setup steps](#high-level-environment-setup-steps)
  - [Create serverless-ENVIRONMENT.yml](#create-serverless-environmentyml)
  - [Create staging GitHub action](#create-staging-github-action)
  - [Initial push](#initial-push)
  - [Finalize staging setup and test](#finalize-staging-setup-and-test)
  - [Create production configuration](#create-production-configuration)
  - [Deploy to production](#deploy-to-production)
  - [Finalize Production CI/CD](#finalize-production-cicd)
  - [Removing Serverless-nextjs](#removing-serverless-nextjs)
  - [Inspired by](#inspired-by)

## Overview

This is an example of how to setup ci/cd for a project using the awesome serverless nextjs component. GitHub Actions are used to create a basic ci/cd workflow where commits to main branch deploy a staging environment while creating a release by tagging a commit deploy the production environment.

The implementation is not for the faint of heart. Bootstrapping your project's resources requires manual steps and "priming the pump". The setup is not ideal for sure. However, once you get past the initial deployment, you will find the ci/cd to just work.

The steps below take you through the exact steps to deploy this repo to Lambda@Edge. Hopefully you can follow along. To cut to the chase simply clone this repo for as a starter and filling in evn variables as needed.

It is presumed that you know how to use git, GitHub, GitHub Actions, aws, the aws cli and understand what the Serverless Nextjs Component is and how it works. This process also requires that you have a domain managed by aws (Route 53) that is the targeted deployment domain.

If you're not scared away yet, please follow along as you are guided through setting up your serverless-nextjs project for automated ci/cd with Github Actions.

Note: this is likely a temporary solution until serverless-nextjs supports serverless framework component v2.

## Getting started

Our goal is to deploy a project generated with create-next-app and serverless nextjs to staging and production environments simply by checking into the main branch and tagging a commit with a version number.

First, create a new repository on GitHub. After all we intend to deploy staging and production environments automatically from Github ;-)

Be sure to clone the repository to your local development environment.

Next, create a new nextjs application.

```bash
npx create-next-app
✔ What is your project named? serverless-nextjs-github-ci-cd
✔ Pick a template › Default starter app
```

Once installation is complete navigate to the project's home directory.

```bash
cd serverless-nextjs-github-ci-cd
```

npm is used throughout this guide so we need to do a little cleanup to remove yarn.lock and create a package-lock.json file

```bash
rm yarn.lock && npm install --package-lock-only
```

_The ci process requires that the package-lock.json file is tracked. Please ensure you commit package-lock.json to your repository._

now, confirm the nextjs development server is working.

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) with your browser to see the result.

Congratulations, you now have a working local development environment.

Commit changes.

## Install serverless

Installing the serverless framework as a development dependency results in advantages later on with the ci process.

```bash
npm install serverless@latest --save-dev
```

## Create S3 serverless assets storage bucket

An S3 bucket is needed to store deployment configurations for the serverless-nextjs component. A single bucket can be used for all serverless-nextjs project. S3 bucket names must be unique within all of AWS. **If you check out this repo YOU MUST CHANGE THE NAME OF THE s3 bucket. You will get an error if you do not.**

Create an S3 bucket named `<YOUR_AWS_USERNAME>-serverless-state-bucket`. Select the settings you'd wish. In general the default options are good. Be sure that "Block all public access" is checked.

## Setup GitHub secrets

GitHub actions require your AWS key and secret. These need to be added to your GitHub account as environment variables.

Log in to your GitHub account and navigate to Settings. Select Secrets in the left sidebar. Click New Secret to add `AWS_ACCESS_KEY_ID`. Click New Secret to add `AWS_SECRET_ACCESS_KEY`.

## High level environment setup steps

In our scenario there are 3 environments:

- dev: local development
- staging: non production preview of dev environment deployed to AWS
- prod: the "live" site that users access on AWS

Staging and prod environments require their own serverless config file. Setting up a new environment has a similar recipe to the steps below.

- create a serverless-ENVIRONMENT.yml file with environment configuration
- create a GitHub action for the environment
- comment out download of serverless state from S3 in GitHub action (does not exist until after first deploy)
- execute a GitHub Action which runs the serverless command
- uncomment out download of serverless state from S3 in GitHub action (now the .serverless directory exists in the S3 bucket so future deployments can download the directory)

## Create serverless-ENVIRONMENT.yml

Here is a sample "staging" serverless configuration file.

```yaml
# serverless-staging.yml
name: staging-your-site-name

staging-your-site-name:
  component: '@sls-next/serverless-component@latest'
  inputs:
    bucketName: staging-your-site-name-s3
    description: "Lambda@Edge for staging-your-site-name"
    name:
      defaultLambda: staging-your-site-name-lambda
      apiLambda: staging-your-site-name-lambda
    domain: ['staging-your-site-name', 'bobhall.net']
    publicDirectoryCache: false
    runtime:
      defaultLambda: 'nodejs12.x'
      apiLambda: 'nodejs12.x'
```

## Create staging GitHub action

```yaml
# .github/workflows/staging.yml
#
# GitHub Action for Serverless NextJS staging environment
#
name: Deploy staging-your-site-name
on:
  push:
    branches: [main]
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - uses: canastro/copy-file-action@master
        with:
          source: 'serverless-staging.yml'
          target: 'serverless.yml'

      - uses: actions/setup-node@v2-beta
        with:
          node-version: '12'

      - name: Install dependencies
        run: npm ci

      # - name: Run tests
      #   run: npm run test:ci

      - name: Serverless AWS authentication
        run: npx serverless --component=serverless-next config credentials --provider aws --key ${{ secrets.AWS_ACCESS_KEY_ID }} --secret ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      # - name: Download `.serverless` state from S3
      #   run: aws s3 sync s3://bhall2001-serverless-state-bucket/your-site-name/staging/.serverless .serverless --delete

      - name: Deploy to AWS
        run: npx serverless

      - name: Upload `.serverless` state to S3
        run: aws s3 sync .serverless s3://bhall2001-serverless-state-bucket/your-site-name/staging/.serverless --delete
```

## Initial push

The first push after setting up the GitHub Action our serverless state bucket does not have a .serverless directory (required when deploying your website). The .serverless directory is copied to the S3 bucket on the first run of our process. Be sure the serverless-staging.yml file has these lines commented out for the initial commit/push.

Commit your changes and push to GitHub. The push triggers your workflow to come to life executing the ci/cd steps.

_I've found that when using bucketName the initial deployment may fail with an error about the bucket not having transfer acceleration. If this happens, simply re-run the full GitHub action in the GitHub ui._

If all goes well, after about 15 minutes, your staging environment is available. The initial deploy takes some time to set up and for the new endpoint to become available. Now that everything is setup, deployments go much faster.

## Finalize staging setup and test

Once the site is available at the endpoint, remove comments in `.github/workflows/staging.yml` to enable downloading the .serverless directory from the S3 bucket during next the ci/cd process.

Commit the changes and push to Github.

Congratulations! You now have a ci/cd process for staging when commits are made to the main branch.

Going forward the .serverless directory is downloaded as a step in your ci/cd process.

## Create production configuration

Sample production serverless configuration. An advantage of separate files is that we are able to have different configurations for environments. In the setup below, the publicDirectoryCache is set to true.

```yaml
# serverless-prod.yml
name: prod-your-site-name

prod-your-site-name:
  component: '@sls-next/serverless-component@latest'
  inputs:
    bucketName: prod-your-site-name-s3
    description: "Lambda@Edge for prod-your-site-name"
    name:
      defaultLambda: prod-your-site-name-lambda
      apiLambda: prod-your-site-name-lambda
    domain: ['prod-your-site-name', 'bobhall.net']
    publicDirectoryCache: true
    runtime:
      defaultLambda: 'nodejs12.x'
      apiLambda: 'nodejs12.x'
```

Sample production GitHub Action

```yaml
# .github/workflows/prod.yml
#
# GitHub Action for Serverless NextJS production environment
#
name: Deploy prod-your-site-name
on:
  push:
    tags: # Deploy tag (e.g. v1.0) to production
      - 'v**'
jobs:
  deploy-prod:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - uses: canastro/copy-file-action@master
        with:
          source: 'serverless-prod.yml'
          target: 'serverless.yml'

      - uses: actions/setup-node@v2-beta
        with:
          node-version: '12'

      - name: Install dependencies
        run: npm ci

      - name: Serverless AWS authentication
        run: npx serverless --component=serverless-next config credentials --provider aws --key ${{ secrets.AWS_ACCESS_KEY_ID }} --secret ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      # - name: Download `.serverless` state from S3
      #   run: aws s3 sync s3://bhall2001-serverless-state-bucket/your-site-name/prod/.serverless .serverless --delete

      - name: Deploy to AWS
        run: npx serverless

      - name: Upload `.serverless` state to S3
        run: aws s3 sync .serverless s3://bhall2001-serverless-state-bucket/your-site-name/prod/.serverless --delete
```

## Deploy to production

First, we need to commit these changes and push to the main branch. This will trigger the staging deploy. Wait for this deployment to complete.

Using GitHub web ui, locate the "Release" panel in the right sidebar in the "Code" tab of your repository. Click "Create a new release" in the sidebar.

Release version numbers must follow the pattern of "vX.Y.Z" (without quotes, v is required). Tag Version as v1.0.0. Add a title and/or a description to the release.

_The next step will deploy your project to your "production" environment. DO NOT DO THE NEXT STEP UNTIL YOU ARE READY TO DEPLOY TO PRODUCTION._

Click the "Publish release" button.

Congratulations. You have just deployed your serverless nextjs website to the world!

## Finalize Production CI/CD

We have one last step to do before we can call it a victory. Uncomment the code to enable downloading the production .serverless directory from the S3 bucket in `.github/workflows/prod.yml`.

```yaml
- name: Download `.serverless` state from S3
  run: aws s3 sync s3://bhall2001-serverless-state-bucket/your-site-name/prod/.serverless .serverless --delete
```

Commit these changes to main. What for the staging ci/cd to complete. Then create release v1.0.1 to confirm the final workflow deploys successfully. To create a new release click the Release title in the sidebar then the "Draft New Release" button.

## Removing Serverless-nextjs

Serverless remove does not work correctly with the serverless-nextjs component. This is not related to Github actions ci/cd but is an issue with the component itself.

If you need to remove a nextjs application deployed with serverless-nextjs component, follow these steps:

1. `serverless remove` from command line.

1. log in to aws console

1. navigate to CloudFront and disable the Distribution for the deployed serverless-nextjs component (be sure to select the correct distribution if you have many).

1. wait about 5 minutes

1. delete the CloudFront distribution for the deployed serverless-nextjs component (again make sure because there's no going back if you delete an incorrect distribution)

1. wait about 15 minutes. During this time your CloudFront distribution is deleted from edge locations

1. Now you can delete the Lambda function(s) created by the serverless-nextjs component. NOTE: If you get an error that the lambda function can not be deleted, you need to wait longer. The time does vary to delete all the CloudFront distributions. I find 15 minutes is an average time. There may be 1 or 2 lambda's -- one for content, one for api.

You have now removed the serverless-nextjs deployment from aws.

## Inspired by

[Serverless Nextjs Component](https://github.com/serverless-nextjs/serverless-next.js#serverless-nextjs-component)

[Serverless deploy with state management in S3 ](https://gist.github.com/hadynz/b4e190e0ce10e5811cb462920a9c678f)

[Github Actions - Deploy Serverless Framework (AWS)](https://gist.github.com/maxkostinevich/9672966565aeb749d41e49673be4e7fa)

[Copy File GitHub Action](https://github.com/marketplace/actions/copy-file)
