+++
date = "19 Apr 2017"
title = "Automatic deployment to Google Cloud Functions"
draft = false
url = "/posts/automatic-deployments-to-google-cloud-functions-with-travis-ci"
tags = ["node", "google cloud functions", "serverless", "deployment", "paas"]
image = ""
comments = true
share = true
menu= ""
author = "Michael DeRazon"
+++

In this post I'll show you how to achieve a Heroku-like push-to-deploy capability on Google Cloud Functions.

If you don't know Google Cloud Functions (GCF), it's a serverless architecture for running services on Google's services. It's equivalent to AWS Lambda.
You can read about it [here](https://cloud.google.com/functions).

### Goal
Pushing any code to the `master` branch of our Github repository should automatically trigger a deploy of our new code to GCF.

### Overview
We'll need an intermediary service to listen for code changes from Github and pull that code and deploy it to GCF.

We'll use a continuous integration service (CI) for this. In this example, I'll use [Travis-CI](https://travis-ci.com) but any other CI can be used, the instructions will be a bit different though.

\* note: Travis is free for public GH repos and costs money for private repos. This tutorial is relevant for both the free and paid versions.

#### Flow

![](/images/posts/gcf-diagram.svg)

1. Code gets pushed to a remote Github repo.
2. Github triggers a build in Travis.
3. Travis triggers a deploy on GCF.
4. GCF pulls code from Google Cloud Source Repositories and deploys it.

### Set up Travis
I won't go into the details on how to connect Travis and Github, it's pretty straightforward.
Assuming this is a Node.js project (as for now, GCF only supports Node), we'll use the following `.travis.yml` file:
<script src="https://gist.github.com/mderazon/5a5b50d92f4c4adaf44d5f49dd95d0ef.js"></script>

#### Google service account key
We need to be able to log in to our Google Cloud project and deploy a function from the build machine.

To do that, we first need to obtain a service account key from Google IAM. You can do it from [here](https://console.cloud.google.com/iam-admin/serviceaccounts/project). You'll need to give the service account the project *Editor* role (read in their [doc](https://cloud.google.com/functions/docs/concepts/iam)). This is so that our build script can access the GCF project and deploy functions.

Download the service account key in json format and put it in the root folder of your project. Name it `gcloud-service-key.json`.

This file allows anyone access to our GCP project so we don't want to commit it to our code base and push to Github. We'll have to encrypt it first. Luckily, Travis supports this. See their [tutorial](https://docs.travis-ci.com/user/encrypting-files/) on how to encrypt files.
If you have travis CLI, you can encrypt it with:
```
$ travis encrypt-file gcloud-service-key.json
```
Then you should be given a line to put into the build script. Something like:
```
openssl aes-256-cbc -K $encrypted_0a6446eb3ae3_key -iv $encrypted_0a6446eb3ae3_key -in gcloud-service-key.json.enc -out gcloud-service-key.json -d

```
(This is just an example, use the one from the `travis encrypt-file` command)

You should add this line in place of the line in the build script ([L33](https://gist.github.com/mderazon/5a5b50d92f4c4adaf44d5f49dd95d0ef#file-travis-yml-L33))

So now you should have two files in your project root:

1. gcloud-service-key.json
2. gcloud-service-key.json.enc

You can go ahead and delete the original service key `gcloud-service-key.json` now so that we don't commit it by accident.

#### Google Cloud Source Repository
Cloud Functions can't pull code directly from Github. Instead, code gets pulled from [Cloud Source Repository](https://cloud.google.com/source-repositories/). Read the [instructions](https://cloud.google.com/source-repositories/docs/connecting-hosted-repositories) on how to connect our Github project to a GCloud repository. The GCloud repo will effectively mirror our Github repo so that the code in the two will be identical at all times.

Once you create a link between your Github and GCloud repositories, replace *PROJECT* and *REPO* in our build script ([L36](https://gist.github.com/mderazon/5a5b50d92f4c4adaf44d5f49dd95d0ef#file-travis-yml-L36), [L42](https://gist.github.com/mderazon/5a5b50d92f4c4adaf44d5f49dd95d0ef#file-travis-yml-L42)) with your values.

### Push to deploy
Now all left to do is to push code your Github repo and check Travis logs to see that everything runs smoothly.
