# Intro

This will be a quck tutorial on how to setup CI/CD pipelines on the Azure DevOps platform. The intuition of the *validation* & *deploy* pipelines is to...

1. **trigger** the validation pipeline using the Azure DevOps branch polices, when a **Pull Request** is created
2. **validate a package** running all **local tests**
3. keep track of the **deploy id** by creating a new branch on the git repo that you are using, and saving it to a new file
4. utilizing the deploy pipeline, find the deploy id and **quick deploy** the validated package

## Requrments

1. **openssl** you will need it for the generation of public & private keys.
2. **Azure account** (duh...)
3. **Git repo** on Azure (you can host it in a different location also)

___

# Configuration

You can follow the steps below to configure the *validation* and *deploy* pipelines to Azure.

## Step 1: Create certificate

We will use a self-signed certificate to cofigure the deployment app in Salesforce. In your terminal run the following commands to generate the files.

```shell

openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out server.key
openssl req -new -key server.key -out server.csr
openssl x509 -req -sha256 -days 365 -in server.csr -signkey server.key -out server.crt
```

> These are sensitive information, try not to expose them...

## Step 2: Setup the Connected App

Using the slef-signed key, we will create a *Deployment* connected app on the targer Sf org where the pipelines should run. This will authorize the pipelines to connect & deploy packages to the target org. You should also consider to create a new custom profile named *Deployment* with permissions to deploy packages. A user should also be created to be used on the authorization of the pipeline.

1. Login to the target org with an Admin user
2. Navigate to Setup
3. Type on the quick search `App Manager` and click on **App Manager**
4. Click on **New Connected App**
   - Connected App Name: `Deployment app`
   - Contact Email: email address of the maintainer
   - OAuth Settings: mark it as checked
   - Callback URL: `http://localhost:1717/OauthRedirect`
   - Use digital signatures: mark it as checked
     - select the `server.crt` file
   - OAuth scopes
     - Access and manage your data (api)
     - Access your basic information (id, profile, email, address, phone)
     - Perform requests on your behalf at any time (refresh_token, offline_access)
     - Provide access to your data via the Web (web)
   - Require Secret for Web Server Flow: mark it as checked
   - Click on **save**
5. On the newly created app view, click on **Manage**
6. Permitted Users: `Admin approved users are pre-authorized` then **save**
7. On *Profiles* click on **Manage Profiles** and add the *Deployment* profile (or *System Administratior*)
8. Create a new user named *Deployment* and assigne the *Deployment* profile (or *System Administratior*)

## Step 3: Setup the Azure pipelines

At this point it expected that you have setup the git repo, branches etc. The git user that will be used should have access to the repo and can be a dummy account. In Azure:

1. Click on **Pipelines** and then click on **Create Pipeline**
2. Select the following settings (if you have your cade on a different repo, you should manualy link it here)
   - Where is your code? -> *Azure Repos Git*
   - Select your repository -> select your repository
   - Configure your pipeline -> *Starter Pipeline*
3. Overwrite the YAML with the `deploy.yml` or `validation.yml` and click save
4. On *Variables* create the following:
   - salesforceDevOrgUserName: type the username of the *Deployment user*
   - salesforceDevOrgClientId: client id from the *Deployment connected app*
   - salesforceDevOrgInstanceURL: `https://login.salesforce.com` (or `https://test.salesforce.com` in case the target repo is a sandbox)
   - gitName: type the git username
   - gitEmail: type a git email of the above user
   - serverKey: copy and paste the contents of the *server.key* file

### YAML notes

This will be a very quick reference/highlight on the YAML components, for more info please refer to the [oficial documentation](https://docs.microsoft.com/en-us/azure/devops/pipelines/?view=azure-devops).

- **Triggers** need to be specified to indicate when should the pipeline run
  - **none** a pipeline with no CI trigger
  - **branches** (include/exclude) will run the pipeline if a commit is pushed to the included branches
  - **paths** (include/exclude) will run the pipeline if a file is changed under the include paths
  - **pr** the pipeline will run when a Pull request is opened (& when a new commit on the source branch is pushed) **can be specified in the branch polices as a build validation**

> more about triggers can be found in [Build Azure Repos Git or TFS Git repositories](https://docs.microsoft.com/en-us/azure/devops/pipelines/repos/azure-repos-git?view=azure-devops&tabs=yaml#ci-triggers)

- With **Steps** you can spilt the run of your pipeline in segments using bash **script**. Make use of the Salesforce CLI commants to impliment your logic.

## Step 4: Add branch polices

> This is applicable only on Azure DevOps repos

Branch polices are crusial for the CI pipelines to opperate correctly. We want to block any users from directly pushing commits to all branches except the feature branches that they will develop on. Also, we can specify some hard reqirments for the Pull Requests along side the **Build Validation** to trigger the *validation* pipeline.

Navigate on the Azure DevOps and click on **Branches**

1. Click on the 3 dots on the right side of the branch you want to setup
2. Click on the **Branch polices**
3. *Optionally* set branch polices here (recomented)
4. Scroll down to the **Build validation** section
   1. Click on the + button to add a new pipeline run
   2. Select one of the Validation pipelines that you have already created on step 3
   3. Select Trigger **Automatic**
   4. *Optionally* Policy reqirment to **Reqired** (recomented)
   5. *Optionally* you can also set an expiration of the build here
   6. Click **save**

That is all! You are now a proud owner of a Salesforce repo with CI/CD. Create a new branch, push some changes onto it, and create a PR to test everything. 