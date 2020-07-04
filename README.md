# serverless-nextjs-github-ci-cd

## WORK IN PROGRESS NOT COMPLETE IF YOU SEE THIS MESSAGE

This is an opinionated view of how to setup a [Next.js](https://nextjs.org/) project bootstrapped with [`create-next-app`](https://github.com/vercel/next.js/tree/canary/packages/create-next-app) using [Serverless Nextjs Component](https://github.com/serverless-nextjs/serverless-next.js) with Github Actions for ci/cd.

## Contents

- [Motivation](#motivation)
- [Overview](#overview)
- [Getting started](#getting-started)

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
rm yarn.lock && npm i --package-lock-only
```

_The ci process requires that the package-lock.json file is tracked. Please ensure you commit package-lock.json to your repository._

now, confirm the nextjs development server is working.

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) with your browser to see the result.

Congratulations, you now have a working local development environment.

Commit changes to your repository.
