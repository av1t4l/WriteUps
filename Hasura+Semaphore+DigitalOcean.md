## Hasura CI/CD with Semaphore and Digital Ocean

# Introduction
***Problem:** 
I want to use Hasura for the backend of my app. 
I want proper version control.
I want a CI/CD pipeline to deploy my Hasura backend*.

If you are thinking the same, then this is for you. I scoured Google for this tutorial for days to no avail (because its such a specific stack), so after figuring out this process for myself, I decided to make life easier for all of you readers. 

In this tutorial you will learn how to create a basic CI/CD Pipeline using Semaphore, Digital Ocean Droplets and Hasura. 


# Prerequisites
The following a necessary to follow along with this tutorial:
- A Semaphore Account -- pay per use account, I chose linux `e1-standard-2` 
- A Digital Ocean account -- pay per use for the droplets
- A github account -- can be the free tier, I am assuming you have basic git knowledge 
- Optional: A domain -- this will probably incur a small fee 


## Create Hasura Droplet using Hasura Marketplace
Log into Digital Ocean and "Create" a new droplet. Select the Marketplace tab -> See All Marketplace Apps and search for "Hasura".

Once you have selected the Hasura droplet, configure the droplet how you would like. 
I'm using: 
- 5$ a month
- SSH keys -- I chose a pre existing SSH key because it was already set up.

If you are creating key, don't use as passphrase as we will need this key for automated deployment later -- Not ideal, I know, but I have yet to find a workaround 
https://docs.hasura.io/1.0/graphql/manual/guides/deployment/digital-ocean-one-click.html
Follow this for the majority to configure your droplet 

**make sure you configure domain and SSL here

## Set Up Git Repo
If you don't have it already, create a new/blank git repository for the project. 

Because the droplet has Hasura already installed, we don't have to worry about setting it up. 

SSH into your droplet using the ip on the digital ocean dashboard. In the previous step you should have set up SSH keys and it should not require a passphrase. 
```
    ssh root@droplet-ip
```

Once in the droplet, run the following commands to clone the git repository. You man need your github credentials. 
```
    cd /etc/hasura
    git clone
```

Then save your github credentials, then pull again to make sure the credentials were saved
```
    git config --global credential.helper store
    git pull
    git pull
```

If you are following good development practices, then you will want to create branches and now is the time to do it. I have a my staging and master branches checked out on my droplets. 

Make sure the Hasura CLI is installed on the droplet by trying to run the `hasura` command. 
If no help screen appears, install using the docs: 
https://docs.hasura.io/1.0/graphql/manual/hasura-cli/install-hasura-cli.html


<!-- 2. Set up the git repository on the droplet 
    - in /etc/hasura `run git clone` -- if you have an existing repo 
    - save your github username and password `git config --global credential.helper store` then `git pull` then they should be saved, and repeat to check

    set branches here if you need to branch https://confluence.atlassian.com/bitbucket/branching-a-repository-223217999.html

    make sure hasura CLI is installed -->

## Configure Hasura for Migrations 
The Hasura documentation on this topic is very comprehensive, so follow these steps: 
https://docs.hasura.io/1.0/graphql/manual/migrations/new-database.html

Making sure you: 
- disable the console = no need for it. 
    *We are using migrations so the changes should be made locally and pushed to git to be deployed on the deployed version* 
- install the hasura CLI -- to run the migrations 
- run `hasura init` as it creates the project and migrations folder

NOTE: Don't commit config file to git as this will vary between deployments. 

## Set up Semaphore
Semaphore is my CD/CD platform of choice, mainly because it supports iOS, which is a front end component of this project. It uses block structures to configure pipelines which seems straight forward but there can be a bit of a learning curve. 

Our pipeline is going to consist of:
- Connecting to our git repo
- SSHing into our droplet
- running the migrations

Start by connecting your git repository to Semaphore as a project. Choose single job, then quit as we need to add a secret before we can finish configuring the pipeline. 

## Set up Secrets/SSH keys for Semaphore
You will need to have Semaphore CLI set up on your local computer as we are going to upload a copy of our Droplet's SSH key to semaphore. 
Follow theses docs to install the CLI:
https://docs.semaphoreci.com/reference/sem-command-line-tool/
   
- Once the CLI is installed, In the project, click on the `>_` in the top left of the screen, then expand the "Install CLI and Connect to ...". Paste that command into your local terminal.

- Once the CLI is configured and project set up copy the SSH key to Semaphore and create a secret called hasura_secret
```
    sem create secret hasura_secret -f ~/.ssh/id_rsa:~/.ssh/id_rsa_hasura 
``` 
*`~/.ssh/id_rsa` -- this is the default location of the droplets secret key, keep note if you changed it.*


## Set up the workflow 
Select the branch you would like deployed and click on 'Edit Workflow'. 

The first block is connecting to your repo and where will you run your tests. Lets just focus on the deployment part which will be a 'promotion block'.

Semaphore uses 'promotions' to deploy. These can be manual or automatic. Lets focus on manual promotions.

Add your first promotion, click on block 1, job 1
Rename job and promotion by clicking on them 
In prologue add the following to set up the SSH key we previously uploaded:
```
    sem-version node 10.13.0
    chmod 0600 ~/.ssh/<name_of_sshkey>
    ls ~/.ssh/
    ssh-add ~/.ssh/<name_of_sshkey>
    ssh-keyscan -H -p 22 <ip_of_droplet> >> ~/.ssh/known_host
```

In job, add this command to perform the SSH connection to the box AND run the command to pull the repo, and run the migrations.
```
    ssh -o "StrictHostKeyChecking no" root@<ip_or_domain_name_of_droplet> "cd /etc/hasura/<project_name> && git pull origin <branch> && hasura migrate apply --skip-update-check"
```
In secrets drop down, select the hasura secret key 
Then run the workflow, you will need to manually promote the deployment. If it all has deployed correctly there should be no errors. Although the error logs, found by clicking the red arrow should be enough debug the situation. 


## Finally 
With the pipeline successfully set up, all changes made locally, committed and pushed will be reflected on the remote version. The deployment will run on every push to the branch you configured in Semaphore. 

To test this pipeline:

- Download the hasura docker container
- Clone the git repo on your local machine
- Create a `config.yaml` file in the same directory as `/migrations` that contains just this line
```
    endpoint: http://localhost:8080
```
- In the directory that contains `docker-compose.yaml` run `docker-compose up -d`
- Then run `hasura console`. This will open a page on the web browser, any changes made to the schema in this console will be added to the migrations folder.
- Make a change, then 
```
    git commit -m "..." && git push 
```
- This should prompt Semaphore to run the workflow to deploy, accept the promotion, then check the change was propagated to the server

Hopefully this guide was comprehensive to get this workflow working for you too, and spared you the many hours I wasted trying to find a writeup on this particular stack!

