# serverless-nextjs-github-ci-cd

## WORK IN PROGRESS NOT COMPLETE IF YOU SEE THIS MESSAGE

This is an opinionated view of how to setup a [Next.js](https://nextjs.org/) project bootstrapped with [`create-next-app`](https://github.com/vercel/next.js/tree/canary/packages/create-next-app) using [Serverless Nextjs Component](https://github.com/serverless-nextjs/serverless-next.js) with Github Actions for ci/cd.

## Contents

- [Motivation](#motivation)
- [Overview](#overview)
- [Getting started](#getting-started)
- [Install serverless](#install-serverless)
- [Configuration file](configuration-file)
- [Create S3 serverless assets storage bucket](create-S3-serverless-assets-storage-bucket) -[Create serverless-staging.yml](create-serverless-staging.yml)

## Motivation

There is a desire to implement ci/cd process with environments where checking into a branch in a Github repository performs ci/cd tasks resulting in a staging environment while tagging a commit with a version number (or creating a release in Github) performs ci/cd tasks for a production environment.

## Overview

This is an example of how to setup ci/cd for a project using the awesome serverless nextjs component. Github Actions are used to create a basic ci/cd workflow where commits to master branch deploy a staging environment while creating a release by tagging a commit deploy the production environment.

The implementation is not for the faint of heart. Bootstrapping your project's resources requires manual steps and "priming the pump". The setup is not ideal for sure. However, once you get past the initial deployment, you will find the ci/cd to just work.

The steps below take you through the exact steps to deploy this repo to Lambda@Edge. Hopefully you can follow along. To cut to the chase simply clone this repo for as a starter and filling in evn variables as needed.

If you're interested, please follow along as you are guided through setting up your serverless-nextjs project for automated ci/cd with Github actions.

Note: this is likely a temporary solution until serverless-nextjs supports serverless framework component v2.

## Getting started

Our goal is to deploy a project generated with create-next-app and serverless nextjs to staging and production environments simply by checking into the master branch and tagging a commit with a version number.

First, create a new repository on Github. After all we intend to deploy staging and production environments automatically from Github ;-)

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
npm install serverless@1.74.1 --save-dev
```

_note: the repo pins version numbers that are known to work_

## Configuration file

create next.config.js at the root directory

```bash
touch next.config.js
```

with

```javascript
module.exports = {
  target: 'serverless',
};
```

## Create S3 serverless assets storage bucket

An S3 bucket is needed to store deployment configurations for the serverless-nextjs component. A single bucket can be used for all serverless-nextjs project.

Create an S3 bucket named `<YOUR_AWS_USERNAME>-serverless-state-bucket`. Select the settings you'd wish. In general the default options are good. Be sure that "Block all public access" is checked.

## Setup Github secrets

Github actions requires your AWS key and secret. These need to be added to your Github account as environment variables.

Log in to your Github account and navigate to Settings. Select Secrets in the left sidebar. Click New Secret to add `AWS_ACCESS_KEY_ID`. Click New Secret to add `AWS_SECRET_ACCESS_KEY`.

## High level environment setup steps

In our scenario there are 3 environments:

- dev: local development
- staging: non production preview of dev environment deployed to AWS
- prod: the "live" site that users access on AWS

staging and prod environments require their own serverless config file. Setting up a new environment has a similar recipe to the steps below.

- create a serverless-[ENVIRONMENT].yml file with environment configuration
- create a github action for the environment
- comment out download of serverless state from S3 in Github action (does not exist until after first deploy)
- run the serverless command
- uncomment out download of serverless state from S3 in Github action (now it exists so future deployments can download from the S3 bucket)

## Create serverless-[environment].yml

Here is a sample "staging" serverless configuration file.

```yaml
# serverless-staging.yml
name: staging-your-site-name

staging-your-site-name:
  component: serverless-next.js@1.14.0
  inputs:
    bucketName: staging-your-site-name-s3
    description: '*lambda-type*@Edge for staging-your-site-name'
    name:
      defaultLambda: staging-your-site-name-lambda
      apiLambda: staging-your-site-name-lambda
    domain: ['staging-your-site-name', 'bobhall.net']
    publicDirectoryCache: false
    runtime:
      defaultLambda: 'nodejs12.x'
      apiLambda: 'nodejs12.x'
```

## Create staging Github action

```yaml
# .github/workflows/staging.yml
#
# Github Action for Serverless NextJS staging environment
#
name: Deploy staging-your-site-name
on:
  push:
    branches: [master]
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

The first push after setting up the Github Action your serverless state bucket will not have a .serverless directory (required when deploying your website). The serverless-staging.yml file should have these lines commented out for the initial commit/push.

Commit your changes and push to Github. The push triggers your workflow to come to life to execute the ci/cd steps.

\_I've found that when using bucketName the initial deployment may fail with an error about the bucket not having transfer acceleration. If this happens, simply re-run the full github action in the Github ui.

If all goes well, after about 15 minutes, your staging environment is available. The initial deploy takes some time to set up and for the new endpoint to become available. Now that everything is setup, deployments go much faster.

## Finalize staging setup and test

Once the site is available at the endpoint, remove the comments the lines in `.github/workflows/staging.yml` to enable downloading the .serverless directory from the S3 bucket during the ci/cd process.

Commit the changes and push to Github.

Congratulations! You now have a ci/cd process for staging when commits are made to the master branch.

## Create production configuration

Sample production serverless configuration. This allows you to have different configurations for staging/production. In the setup below the publicDirectoryCache is set to true.

```yaml
# serverless-prod.yml
name: prod-your-site-name

prod-your-site-name:
  component: serverless-next.js@1.14.0
  inputs:
    bucketName: prod-your-site-name-s3
    description: '*lambda-type*@Edge for prod-your-site-name'
    name:
      defaultLambda: prod-your-site-name-lambda
      apiLambda: prod-your-site-name-lambda
    domain: ['prod-your-site-name', 'bobhall.net']
    publicDirectoryCache: true
    runtime:
      defaultLambda: 'nodejs12.x'
      apiLambda: 'nodejs12.x'
```

Sample production Github Action

```yaml
# .github/workflows/prod.yml
#
# Github Action for Serverless NextJS production environment
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

## Deploy to Production

First, we need to commit these changes and push to the master branch. This will trigger the staging deploy. Wait for this deployment to complete.

Using Github web ui, locate the "Release" panel in the right sidebar while in "Code" of your repository. Click "Create a new release"

Release version numbers must follow the pattern of "vX.Y.Z" (wihtout quotes and v is required). Tag Version as v1.0.0. Add a title and/or a description to the release.

_The next step will deploy your project to your "production" environment. DO NOT DO THE NEXT STEP UNTIL YOU ARE READY TO DEPLOY TO PRODUCTION._

Click the "Publish release" button.

Congratulations. You have just deployed your serverless nextjs website to the world!

## Finalize Production CI/CD

We have one last step to do before we can call it a victory. Uncomment the code to enable the production .serverless directory from the S3 bucket in `.github/workflows/prod.yml`.

Commit these changes to master. What for the staging ci/cd to complete. Then create v1.0.1 release to confirm the final workflow deploys successfully.
