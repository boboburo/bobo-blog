---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "Deploy Data with Gcloud, Datasette and Github Actions"
subtitle: "A practice run in using Datasette, Github Actions to manually deploy data on gcloud"
summary: ""
authors: []
tags: [gcloud,datasette,github actions]
categories: []
date: 2020-10-12T13:17:34+01:000
lastmod: 2020-10-12T13:17:34+01:00
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

# Intent 

I recently came across the [Datasette](https://docs.datasette.io/en/stable/) library - that aims to simplify the storing and publishing of data. This is something that I have wanted to do - with the eventual aim of building some data apps/products (read Shiny App) that decouples the data storage from the app. 

Datasette has excellent documentation and [this post](https://simonwillison.net/2020/Jan/21/github-actions-cloud-run/) by its creator details how to setup to publish the data to google cloud. 

This post is a repeat of the above with some minor modifications based on my experiences. 

# Setup 

- datasette
- gcloud 
- github actions

## datasette

I setup Datasette locally on my Mac machine to play around with it initally. I have [conda](https://www.anaconda.com/products/individual) installed to manage my python environments. 


```bash
conda create -n bobo-data python=3.8
source activate bobo-data
pip3 install datasette sqlite-utils geojson-to-sqlite
pip freeze > requirements.txt #will use this later to install in glcoud container
```

Datasette has a number of utilities to help push data to SQLite. I followed [this post](https://simonwillison.net/2019/Feb/25/sqlite-utils/) to run first *practice* database. 

> Why SQLite?: The creator of Datasette, Simon Willison - details the reasoning behind using SQLite being; it is very widely used, open source, single file on disk, being single file it is flexible to ship as part of a data application in a Docker container. Check out a full overview video [here](https://www.youtube.com/watch?v=pTr1uLQTJNE).

I created a .sh file with the following command that downloads some data and creates a db with the table meteorties. 

```bash
curl "https://data.nasa.gov/resource/y77d-th95.json" | \
sqlite-utils insert practice.sqlite meteorites - #arguments can be change for csv
```

## gcloud setup

Datasette also facilitates the publish of sqlite db. There are a few options - I have been using glcoud previously so went with that option. 

> gclound run - is a serverless platform, that allows you to deploy stateless conatainerized applications, that has automatic scaling built in. 

You will need to create an account on cloud.google.com and install [Google Cloud SDK](https://cloud.google.com/sdk/) to use CLI.

Google Cloud projects form the basis for creating, enabling, and using all Google Cloud services. In the Google Cloud Console create new project. 

```bash
#check the project is created
gcloud projects list
```

Set configuration for google cloud. *Note: you can set named configurations if you have multiple projects*

```bash
gcloud config set run/region europe-west1
glcoud config set project [Project ID]
```

## publish the dataset 

You need to have billing info associated with Project in Google Cloud for this to work. 

```bash
 datasette publish cloudrun practice.sqlite --service=practice
 ```
 
 And the result dataset is publish [here](https://practice-rx6ijjv5bq-ew.a.run.app/). 
 
## Automate with Github Actions

The last step is to work out how to automate this using Github Actions.

> What is github actions: Github actions are a CI/CD pipelining tool - an alternative to Jenkins or Travis. A lot of CI steps, getting newest code, running a test, logging into a derserver and deploying can be done direct from Github Actions (not still in beta mode). 

So with a little effort it is possible to auctomatically run the above commands from github and *publish* new data to the Gcloud project. 

### Info on Actions. 

The name Github Actions name is a little confusiong - it is a Workflow. The workflow has 4 components. 

- workflow: The flow that is executed.
- job: a workflow consists of one or more jobs. Jobs can run in parallel
- step: a job consists steps or multiple steps.
- action: each step consists actions. 

GitHub workflows are defined in YAML format files that describe which actions or steps need to be executed during the workflow.They are stored in  *.github/workflows/* folder. 

**Getting the YAML format correct is the hardest part :sweat::weary::persevere::sweat_smile: !**

A nice feature of Github Actions is that there are many create actions available that you can build your workflow from. 

Github Actions are 

### Creating Github Action

Firstly I create an .sh of the previous command that grabs the data and creates the sqlite db. I have chance the name of the db. The first line prints out the current path. Sometimes I find this useful when working with VMs/Containers. 

```bash
#!/bin/bash
DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" >/dev/null 2>&1 && pwd )"
echo $DIR

curl "https://data.nasa.gov/resource/y77d-th95.json" | \
sqlite-utils insert practice_with_ga.sqlite meteorites - 
```

I recommend adding each step in turn and testing if it works. 

#### Create the VM

The **on:** parameter allows you to manually trigger the Action, good when you are debugging, but it could equally be event based. 

The **runs:** specifies the VM to run on. 


```yaml
name: Practice Actions
on: [workflow_dispatch]

jobs:
  practice-pub:
    name: Praticing GA

    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: [3.8]
```

There is only 1 step in this workflow, with six steps. Three of which are listed below (Checkout code, Set up python, Install depencies, Grab data). 

Note: Each action does not require a **name:** but each seperate action most be prefereced with **-** for the YAML to formatted correctly. 

Actions *Checkout code* and *Set up Python* **uses:** actions created elsewhere. There is a [marketplace](https://github.com/marketplace?type=actions). 

```yaml
 steps:
    - name: Checkout code
      uses: actions/checkout@v2
    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v1
      with:
        python-version: ${{ matrix.python-version }}
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
    - name: Grab data and create db
      run: |
        . practice/grab-data.sh
```

The last two actions in the step; Setup GCloud and then follow the Datasett commands to publish. 


```yaml
    - name: Set up Cloud Run
      uses: GoogleCloudPlatform/github-actions/setup-gcloud@master
      with:
        version: '275.0.0'
        service_account_email: ${{ secrets.GCP_SA_EMAIL }}
        service_account_key: ${{ secrets.GCP_SA_KEY_practice }}
    - name: Deploy to Cloud Run
      run: |-
        gcloud components install beta
        gcloud config set run/region europe-west1
        gcloud config set project practice-292411
        datasette publish cloudrun practice_with_ga.sqlite --service practice
```

Note the use of *secrets* here. The **GCP_SA_EMAIL** is straightforward. Creating the **GCP_SA_KEY_PRACTICE** from GCloud, navigate to navigated [GCloud IAM & Admin Service Accounts.](https://console.cloud.google.com/iam-admin/serviceaccount)]

Select the project and created a json key. This generates a key for a *service account* associated with the project. Run this command to encode the json into one long string. 

```bash
base64 /path/to/json_file
```
Create a new secret in th github repo where the code resides, by navigating to *Settings -> Secrets -> New Key* and paste the string in. 

Thats it :anguished: if all is order. From the Actions tab you can invoke the **Workflow** manually. You can navigate to the running action and click each drop down to see the following output. 

![image alt text](/actions.png)

## Closing Thoughts 

Here are the two main files associated with this post [grab-data.sh](https://github.com/boboburo/data/blob/master/practice/grab-data.sh) and [Github YAML workflow](https://github.com/boboburo/data/blob/master/.github/workflows/schedule-practice.yml).

I really like the combination of the Datasette, GCloud and Git Actions and think it is a powerful way of the storing and dissemniating small interesting datasets. 

I plan on using this to investigate a simple way of decoupling data from applications in future. 







