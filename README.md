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

This is an example of how to setup ci/cd for a project using the awesome serverless nextjs component. Github Actions are used to create a basic ci/cd workflow where commits to origin/master branch generate a staging environment while creating a release by tagging a commit generates the production environment.

The implementation is not for the faint of heart. Bootstrapping your project's resources requires manual steps and "priming the pump". The setup is not ideal for sure. However, once you get past the initial deployment, you will find the ci/cd to just work.

The steps below take you through the exact steps used to deploy this repo to Lambda@Edge. Hopefully you can follow along. To cut to the chace simply clone this repo for as a starter and filling in evn variables as needed.

If you're interested, please follow along as you are guided through setting up your serverless-nextjs project for automated ci/cd with Github actions.

Note: this is likely a temporary solution until serverless-nextjs supports serverleess framework component v2.

## Getting started

Our goal is to deploy a project generated with create-next-app and serverless nextjs to staging and production environments simply by checking into the master branch and tagging a commit with a version number.

First, create a new repository on Github. Afterall we intend to deploy staging and production environments automatically from Github ;-)

Besure to clone the repository to your local develement environment.

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

npm is used throughtout this guide so we need to do a little cleanup to remove yarn.lock and create a package-lock.json file

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

## Create serverless-staging.yml

We will not create the serverless file used to setup the staging environment. In our scenerio there are 3 environments:

- dev: local development
- staging: non production preview of dev environment depoloyed to AWS
- prod: the "live" site that users access on AWS

Each environment requires it's own serverless config file. In a future step, a github action copies this file to serverless.yml. An advantage of this method is you are able to configure your serverless environments with differently (or not). It's up to you. In this example, publicDirectoryCache is set to false. For the production version we'll set this to true (when we get there).

```bash
touch serverless-staging.yml
```

Here is a sample staging serverless configuration file.

```yaml
# serverless-staging.yml
name: staging-your-site-name

dev-bobhall-net:
  component: serverless-next.js@1.14.0
  inputs:
    bucketname: staging-your-site-name-s3
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
