# Intro

TODO

## Requrments

1. **openssl** you will need it for the generation of public & private keys.
2. **Azure account** (duh...)
3. **Git repo** on Azure (you can host it in a different location also)

# Configuration

## Step 1: Create certificate

We will use a self-signed certificate to cofigure the deployment app in Salesforce. In your terminal run the following commands to generate the files.

```shell

openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out server.key
openssl req -new -key server.key -out server.csr
openssl x509 -req -sha256 -days 365 -in server.csr -signkey server.key -out server.crt
```

> These are sensitive information, try not to expose them...

## Step 2: Setup the Connected App

Using the slef-signed key, we will create a *Deployment* connected app on the targer Sf org where the pipelines should run. This will authorize the pipelines to connect & deploy packages to the target org.

1. Login to the target org with an Admin user
2. Navigate to Setup
3. Type on the quick search `App Manager` and click on **App Manager**
4. Click on **New Connected App**
   - Connected App Name: `Deployment app`
   - Contact Email: email address of the maintainer
   - OAuth Settings: mark it as checked
   - Callback URL: `http://localhost:1717/OauthRedirect` TODO: is that ok?
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
7. On *Profiles* click on **Manage Profiles** and add a *System Administrator* or a custom one that can deploy packages
8. Create a new user named *Deployment* TODO: finish this

## Step 3: Setup the Azure pipelines

At this point it expected that you have setup the git repo, branches etc. In Azure:

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
   - gitName: type the git username TODO: who?
   - gitEmail: type a git email of the above user
   - serverKey: copy and paste the contents of the *server.key* file

> TODO: notes about configuration of the yamls

## Step 4: Add branch polices

> This is applicable only on Azure DevOps repos