
---
- [CI/CD Pipeline Setup – Sparta Test App](#cicd-pipeline-setup--sparta-test-app)
  - [Overview](#overview)
  - [Pipeline Diagram](#pipeline-diagram)
  - [Prerequisites](#prerequisites)
    - [GitHub](#github)
    - [Jenkins](#jenkins)
    - [AWS EC2](#aws-ec2)
  - [STEP 1 – Create Dev Branch Locally](#step-1--create-dev-branch-locally)
  - [STEP 2 – Job 1: CI Test](#step-2--job-1-ci-test)
    - [Job Name](#job-name)
    - [Job Type](#job-type)
    - [Description](#description)
    - [Source Code Management](#source-code-management)
    - [Build Triggers](#build-triggers)
    - [Build Environment](#build-environment)
    - [Build Steps (Shell)](#build-steps-shell)
    - [Post-build Actions (only will be able to add after building job2)](#post-build-actions-only-will-be-able-to-add-after-building-job2)
    - [Console Output](#console-output)
  - [STEP 3 – GitHub Webhook Setup](#step-3--github-webhook-setup)
    - [GitHub → Repo → Settings → Webhooks](#github--repo--settings--webhooks)
  - [STEP 4 – Job 2: CI Merge](#step-4--job-2-ci-merge)
    - [Job Name](#job-name-1)
    - [Job Type](#job-type-1)
    - [General](#general)
    - [Source Code Management](#source-code-management-1)
    - [Method 1: Build Steps (Shell)](#method-1-build-steps-shell)
    - [Method 2: Git Publisher:](#method-2-git-publisher)
    - [Branches](#branches)
    - [Build other projects](#build-other-projects)
    - [Console Output](#console-output-1)
  - [STEP 5 – Job 3: CD Deploy](#step-5--job-3-cd-deploy)
    - [Job Name](#job-name-2)
    - [Job Type](#job-type-2)
    - [General](#general-1)
    - [Source Code Management](#source-code-management-2)
    - [Build Triggers](#build-triggers-1)
  - [STEP 6 – Jenkins SSH Credentials (VERY IMPORTANT)](#step-6--jenkins-ssh-credentials-very-important)
    - [Jenkins → Manage Jenkins → Credentials](#jenkins--manage-jenkins--credentials)
    - [Build Environment](#build-environment-1)
    - [Build Steps (Shell)](#build-steps-shell-1)
    - [Copy Code to EC2](#copy-code-to-ec2)
    - [Console Output](#console-output-2)
  - [STEP 8 – Testing the Pipeline](#step-8--testing-the-pipeline)
    - [Test 1](#test-1)
    - [Test 2](#test-2)
    - [Why we setup the CICD pipeline the way we did?](#why-we-setup-the-cicd-pipeline-the-way-we-did)
  - [Benefits for developers and teams](#benefits-for-developers-and-teams)
    - [Benefits for the organization](#benefits-for-the-organization)
    - [Summary:](#summary)

# CI/CD Pipeline Setup – Sparta Test App

## Overview
**
This project implements a **3-stage CI/CD pipeline using Jenkins, GitHub, and AWS EC2**:

1. **Job 1 – CI Test**

   * Triggered by GitHub webhook on `dev`
   * Installs dependencies and runs tests

2. **Job 2 – CI Merge**

   * Runs only if Job 1 succeeds
   * Merges `dev` → `main`

3. **Job 3 – CD Deploy**

   * Runs only if Job 2 succeeds
   * Copies tested code from Jenkins → EC2
   * Restarts the application on EC2

---

## Pipeline Diagram
```text
Developer Push (dev branch)
        |
        v
GitHub Webhook
        |
        v
Job 1: CI Test
(run npm install & tests)
        |
        v
Job 2: CI Merge
(dev → main)
        |
        v
Job 3: CD Deploy
(copy code to EC2 & restart app)
        |
        v
Live Application Updated
```

---

![CI-CD-Diagram](CI-CD.png)
## Prerequisites

### GitHub

* Repo: `tech515-test-sparta-test-app-cicd`
* Branches:

  * `main`
  * `dev`

### Jenkins

* Jenkins installed and running
* GitHub plugin installed
* NodeJS installed on Jenkins agent

### AWS EC2

* Used ubuntu app:
  * **Node 20 → Ubuntu 22.04**
* Security Group:

  * SSH (22) from Jenkins OR `0.0.0.0/0` (initial testing)
  * App port (e.g. 3000) open
* App dependencies installed (node, npm, pm2)

---

## STEP 1 – Create Dev Branch Locally

```bash
cd tech515-test-sparta-test-app-cicd
git checkout -b dev
git push origin dev
```

---

## STEP 2 – Job 1: CI Test

### Job Name

```
piyusha-job1-ci-test
```

### Job Type

* **Freestyle Project**

### Description
* Testing part of CI with webhook 
* Discard old builds:
  * Strategy: Log rotation: Max # of builds to keep: 5
  * GitHub project: Project url: https://github.com/pchalke/tech515-test-sparta-test-app-cicd/

### Source Code Management

* Git
* Repo URL: your GitHub repo:git@github.com:pchalke/tech515-test-sparta-test-app-cicd.git
* Credentials: piyusha-jenkins-github-key (the key should have read and write permissions)
* Branch to build:

```
*/dev
```

### Build Triggers

✅ **GitHub hook trigger for GITScm polling**

---
### Build Environment
* Provide Node & npm bin/ folder to PATH
* NodeJS Installation
* NodeJS version 20
* Default (~/.npm or %APP_DATA%\npm-cache)


### Build Steps (Shell)

```bash
cd app
npm install
npm test
```
### Post-build Actions (only will be able to add after building job2)
* Build other projects: Projects to build: piyusha-job2-ci-merge
* Trigger only if build is stable
  
* Save
* Apply

> If tests fail → pipeline stops
> If tests pass → Job 2 runs
### Console Output
![Job1-console-output](<untitled folder 2/job1-console-output.png>)

---

## STEP 3 – GitHub Webhook Setup

### GitHub → Repo → Settings → Webhooks

* Payload URL:

```
http://<jenkins-ip>:8080/github-webhook/
```

* Content type: `application/json`
* Events: **Just the push event**
* Save webhook

✅ Pushing to `dev` now triggers **Job 1 automatically**

---

## STEP 4 – Job 2: CI Merge

### Job Name

```
piyusha-job2-ci-merge
```

### Job Type

* **Freestyle Project**
### General 
* Description: merging into the github
* GitHub project: Project url: https://github.com/pchalke/tech515-test-sparta-test-app-cicd/

### Source Code Management

* Git: Repository URL: git@github.com:pchalke/tech515-test-sparta-test-app-cicd.git
* Credentials: piyusha-jenkins-github-key (the key has read and write permissions)
* Branches to build:

```
*/dev
```

---

### Method 1: Build Steps (Shell)

```bash
git fetch origin
git checkout main
git pull origin main
git merge origin/dev
git push origin main
```

> This ensures:

* Only tested code is merged
* No manual merge needed
* Main branch always stable

---
### Method 2: Git Publisher: 
* Push Only If Build Succeeds
* Merge Results
* Force Push

### Branches
Branch to push

```
main
```

Target remote name

```
origin
```
### Build other projects

* Projects to build: piyusha-job3-cd-deploy
* Trigger only if build is stable 
* Save
* Apply

### Console Output
![Job2-console-output](<untitled folder 2/job2-console-output.png>)
## STEP 5 – Job 3: CD Deploy

### Job Name

```
piyusha-job3-cd-deploy
```

### Job Type

* **Freestyle Project**

### General
* Description: copying the already-tested code that exists in the Jenkins workspace to your EC2 instance and run the app
* Discard old builds: Log rotation: Max no of builds to keep: 5

### Source Code Management
* Project url: https://github.com/pchalke/tech515-test-sparta-test-app-cicd/

* Credentials: piyusha-jenkins-github-key (key should have read and write permissions)
* Git
* Branch:

```
*/main
```

---


### Build Triggers

* Build after other projects are built:

```
piyusha-job2-ci-merge
```
* Trigger only if build is stable
---

## STEP 6 – Jenkins SSH Credentials (VERY IMPORTANT)

### Jenkins → Manage Jenkins → Credentials

* Kind: **SSH Username with private key**
* Username: `ubuntu`
* Private Key: paste your **EC2 `.pem` private key**
* ID:

```
piyusha-jenkins-github-key
```

---


### Build Environment

✅ **SSH Agent**

* Credentails:  Specific credentials:

```
ubuntu(tech515-piyusha-aws.pem)
```

---

### Build Steps (Shell)

### Copy Code to EC2

```bash
rsync -av --delete -e "ssh -o StrictHostKeyChecking=no" app ubuntu@172.31.42.123:/home/ubuntu/

#### SSH & Restart App

```bash
ssh -o StrictHostKeyChecking=no ubuntu@172.31.42.123 << EOF
cd /home/ubuntu/app
npm install
pm2 stop all || true
pm2 start app.js
EOF
```

✅ **IMPORTANT**

* No `git clone` on EC2
* Jenkins is the source of truth
* EC2 only receives tested artifacts


### Console Output
![Job3-console-output](<untitled folder 2/job3-console-output.png>)

---

## STEP 8 – Testing the Pipeline

### Test 1

1. Make changes on Sparta app's homepage on `dev`
2. Include date & time
3. Push to GitHub

```bash
git add .
git commit -m "1st change to frontpage"
git push origin dev
```

4. Wait for 1-2 minutes
5. Confirm change appears live

![1st-change-frontpage](<untitled folder 2/1stchange-frontpage-app.png>)


---

### Test 2

Repeat with a new timestamp.

```bash
git commit -am "2nd change to frontpage"
git push origin dev
```

![2nd-change-frontpage](<untitled folder 2/2ndchange-frontpage-app.png>)

---

### Why we setup the CICD pipeline the way we did?
* To catch problems early: the pipeline checks code automatically so bugs are found before they reach users.
* To make builds repeatable: every build follows the same steps so what we test is what we release.
* To speed up delivery: automation lets us release features faster and more often.
* To reduce risk: we run tests and safety checks, and deploy to staging first so production stays safe.
* To keep a record: the pipeline stores which code was built and who approved the release.

## Benefits for developers and teams

* Find bugs sooner — faster fixes and less rework.
* Less manual work — no more manual builds and deployments.
* Faster feedback — you know quickly if your change breaks something.
* Safer merges — bad code doesn't get into the main branch.
* Better teamwork — everyone follows the same process.


### Benefits for the organization

* Faster time to market — features reach customers faster.
* Fewer outages — automated checks reduce production incidents.
* Lower cost of fixes — fewer emergency fixes saves time and money.
* Audit trail — you can show what was released, when, and by whom.
* Scales better — as teams grow, the pipeline keeps releases consistent.


### Summary:
* Automated testing prevents broken code reaching production
* Devs can push frequently without fear
* Main branch always stable
* Faster feedback loop
* Clear separation of concerns:

  * CI → quality
  * CD → delivery
* Industry-standard DevOps workflow

---

