 ---

title: "Build and Deploy static Hugo sites with GitHub Actions"  
date: 2023-04-21T08:00:00+02:00  

draft: false  

description: "Create GitHub Actions to automatically build and deploy a Hugo site via SSH and WireGuard to a private web server."  
keywords: [web, git, github, actions, ssh, wireguard, ci/cd]  

tags: [git, web]  
toc: true

---

To build my site I use the static site generator [Hugo](https://gohugo.io/). The process of editing the Markdown source files and afterwards building the site to deploy it to my website directory, is somewhat old-fashioned: Edit the files with Nextcloud, fire up my VPN, login with ssh and build the site with an alias to deploy it to my directory. Well...

So to adress this issue, I gave the CI/CD system of GitHub Actions a try and want to share my expierences with you.

## Goals

After the completion of this guide, you will be able to:

1. Manage the basic functionality of a git repository on GitHub.
2. Build your static Hugo site with GitHubs CI/CD tool: [GitHub Actions](https://docs.github.com/en/actions/quickstart).
3. Automatically deploy your site onto your webserver via SSH (and optionally with WireGuard).

## Prerequisites

Please take care of the following before you continue:

- GitHub account and the necessary credentials / tools to access it.
- Installed git tools on the machine you want to manage your git repository.
- Up and running sshd.
- Optionally: Configured Wireguard endpoint.

## Set up GitHub related access

### Setup access to the git repository

In order to access your git repository from your local machine, you need to setup a SSH access to GitHub. 

First, lets create SSH keys for your user account. If you already have keys for your user, you can skip this step:

```bash
ssh-keygen -t ed25519
```

You can add a passphrase to make it more secure. The downside is, you have to enter the passphrase every time you use the key, which could be annoying. The decision is yours.

Now you need to export the **public** part of the key-pair and add it to GitHub in order to access *any* public repository on GitHub with `git`.

```bash
cat .ssh/id_ed25519.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBzLyW82qB03GD264sG5/HHOTKdFMvtq2yggxwSxLeui user@localhost
```

Copy the content of your public key and add it here: https://github.com/settings/keys

### Create access for GitHub Actions

The build and deploy process will be done by a small, temporary VM called github-runner.
To deploy the generated website onto your webserver, it needs access to website directory. This will be done by SSH. I highly recommend a seperate system user for that matter.

Create a user. For the purpose of this guide we will call her `github`.
On FreeBSD this will be done with:

```bash
adduser github
```

Make sure the user can access the specific website directory you want to deploy your built website to. Perhaps you have to adjust permission and / or add the user to groups accordingly.
The user needs a home directory for the ssh keys and a login shell for the scp command. Both necessities are the default options for `adduser`.

Login as the newly created user 'github' and create a pair of ssh keys **without** a passphrase:

```bash
ssh-keygen -t ed25519
```

The private key will be added to the secret vault in GitHub Actions later. The public key has to be added to the *authorized_keys* of the user 'github' so the github-runner can login to your webserver via the pubkey authentification:

```bash
cat /home/github/.ssh/id_ed25519.pub >> /home/github/.ssh/authorized_keys
```

## Setup GitHub repository

### GitHub website

First you need to create an empty git repository on the GitHub website. For the purpose of this guide, the repository will be named 'hugo-website'.

Once you have created the repository, you can copy its SSH link. The link can be found in the tab `Code` within the dropdown (green) `Code`. It has the format of:

```bash
git@github.com:[your-github-user/hugo-website.git]
```

### Local repository

Login with your local user account and go to the source of your Hugo website (locally or wherever you are editing your site) to setup your local git repository:

```bash
cd hugo-website
git init
```

You can add any existing files to the repository by typing:

```bash
git add *
```

Optionaly, if you have files you don't want to track, you can add them to `.gitignore`:

```bash
echo "personal/some-files" >> .gitignore
git add .gitignore
```

Commit your changes with the commit message *initial commit* to the local repository:

```bash
git commit -m "inital commit"
```

Now you have to add your GitHub repository so you can push and pull your changes to it. The link can be found in on the GitHub website in your repository as mentioned in the previous section:

```bash
git remote add origin git@github.com:[your-github-user/hugo-website.git]
```

Now select your local repository to be the main branch:

```bash
git branch -M main
```

And last but not least, push your local repository to GitHub. If you have created your SSH keys with a passphrase, you have to enter it every time you access your repository:

```bash
git push -u origin main
```

## GitHub CI/CD with GitHub Actions

GitHub Actions will be responsible to build your Hugo site the same way you would do locally. Afterwards it will deploy it onto your webserver via SSH (scp). Since I can only access my server's sshd from inside my VPN, I have added support to setup a Wireguard interface before deploying. This is purely optional for this guide.

There is a lot of sensitive information involved. To make sure it is not publically available, we make use of [GitHub's encrypted secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets).

GitHub Actions are managed through so called workflows. They describe the job to do, when to be triggered and a hell lot more I still don't know ;)

Once triggered (can be also manually) the workflow is picked up by a github-runner, a small temporary and basic VM. So everything you need, like settings or additional software, has to be taken care of in your workflow. When the job is finished, the VM is gone and the temporary data is lost.

### Creating a workflow file

Every workflow is described in a seperate yaml file located in your repository:

```bash
.github/workflows
```

You can either create your workflow in GitHub's frontend or locally and push it to GitHub's repository afterwards. Lets create the file locally and edit it in your local repository:

```bash
mkdir .github/workflows
code .github/workflows/build-deploy-site.yml
```

### Basic structure of a workflow

The basic layout is the following:

```yaml
    name: Build and deploy Hugo site on my website
    on: [push, workflow_dispatch]  
    jobs:
    deploy:
        name: Build and Deploy
        runs-on: ubuntu-latest
        steps:
```

It consists of the **name** of your workflow. Be creative!

Followed by [workflow triggers](https://docs.github.com/en/actions/using-workflows/triggering-a-workflow) **on**. I want my workflow to run each time I push to my repository and also be able to start it manually.

Next are **jobs** defined. You could do a lot with different jobs which are, for example, dependent on another. Its my first time with CI/CD so I kept it stupidily simple with just one job **deploy**. The job also gets a **name** and which OS the workflow should run on (github-runner). You can basically [select](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners) between Linux, Windows and MacOS and also different versions. You are also able to use your own runners, be it because of technical reasons or licensing concerns. None of those reasons apply to me at the moment.

The **job** consinsts of as many **steps** as you need (or want). Each step runs a set of commands in the VM. It is also possible to use pre-defined or user generated actions already [available at GitHub](https://github.com/marketplace?type=actions).

Lets dive into it!

### Checkout repository

The first step checks out the git repository (on GitHub ofcourse) to the github-runner. There it is avaiable as long as the workflows runs and therefore the VM is active.
Each step gets a name and in this case, uses a predefined GitHub Action.
Optionally, if you use the .GitInfo variable, you should fetch the complete history when doing a git checkout. Thanks goes to [jjameson](https://discourse.gohugo.io/t/problems-with-gitinfo-in-ci/22480)!

```yaml
    - name: Git checkout
        uses: actions/checkout@v3
        # Optional: Fetch all history for .GitInfo and .Lastmod
        with:
            fetch-depth: 0    
```

### Optional: Install git

Hugo's GitInfo variables, `.GitInfo.AuthorDate` in particular, delivered wrong dates back once I have deployed my site. Locally everything worked as expected. Apparently some git config is needed: `git config --global core.quotepath false`

In order to make any changes to the git config on the github-runner, git has to be installed. So I added another step to do that:

```bash
    - name: Install git
        run: |
        sudo apt-get update
        sudo apt-get install -yq --no-install-recommends git
        git config --global core.quotepath false    
```

### Install Hugo

The next step will install and setup Hugo via `apt-get`.

```yaml
    - name: Install Hugo
        run: |
        sudo apt-get update
        sudo apt-get install -yq --no-install-recommends hugo
```

Now Hugo is installed and can build your site. This is also the first time we use a predefined [GitHub variable](https://docs.github.com/en/actions/learn-github-actions/variables): `{{ github.workspace }}` which will point to our checked out repository. 

In order to only deploy the built site later and have it not mixed into the repository, Hugo should deploy the site to the subfolder `htdocs`.

```yaml
    - name: Build Hugo site
        run: hugo -d ${{ github.workspace }}/htdocs --minify
```

### Optional: Setup Wireguard

The next step is optional, but showcases another user generated action and the heavy use of GitHub encrypted secrets for sensitive data. Since my sshd is only accessible inside my VPN, I have to setup Wireguard inside the github-runner to deploy the site via SSH (scp) later.

```yaml
    # Need to install wireguard to access the server (ssh only allowed inside vpn)
    - name: Set up WireGuard
        uses: egor-tensin/setup-wireguard@v1
        with:
        endpoint: '${{ secrets.ENDPOINT }}'
        endpoint_public_key: '${{ secrets.ENDPOINT_PUB_KEY }}'
        ips: '${{ secrets.CLIENT_IP }}'
        allowed_ips: '${{ secrets.ALLOWED_IPS }}'
        private_key: '${{ secrets.CLIENT_PRV_KEY }}'
        preshared_key: '${{ secrets.CLIENT_PRE_KEY }}'
```

The necessary secrets for Wireguard contain the following:

```yaml
	ALLOWED_IPS         - IPs the client will have access to
	CLIENT_IP           - Own IP the client will assign
	CLIENT_PRE_KEY      - Client pre shared key
	CLIENT_PRV_KEY      - Client private key
	ENDPOINT            - Server:Port to reach the Wireguard service
	ENDPOINT_PRIVATE_IP - Private IP of the endpoint 
	ENDPOINT_PUB_KEY    - Public key of wireguard server 
```

### Encrypted GitHub Secrets

In order to use encrypted secrets in your workflow you have to add them in your repository on GitHub in the section **Settings / Secrets and variables / Actions**. They are encrypted after they have been added and only available to the github-runner executing the workflow. You can't view the content of the secret and are only able to overwrite them afterwards, if you want to edit them. There is a lot more information found on [GitHub encrypted secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets?tool=webui).

Before you can deploy your site via SSH, you need to install the SSH keys you created in the beginning of this guide on the github-runner. In order to do that, you have to create two encrypted secrets on GitHub:

`PRIVATE_SSH_KEY` will contain the private key of the user `github` you have created earlier. Copy the content of:

```bash
cat /home/github/.ssh/id_ed25519 
```

in a new secret and name it `PRIVATE_SSH_KEY`. Afterwards you need to put your server's public ssh key (where your website will be deployed) into the `KNOWN_HOSTS` secret in order to make a trusted connection:

```bash
cat /etc/ssh/ssh_host_ed25519_key.pub
```

and past the content into a new secret. You have to add your server's hostname or IP adress in front of the key before saving the secret. It would look somethings similiar to:

```bash
    HOSTNAME_OR_IP ssh-ed25519 PUBLIC_KEY_HASH
```

### Setup SSH access to the webserver

Now the github-runner will setup the private ssh key and the known_hosts file so it can access your server via ssh.

```yaml
    - name: Install SSH Key
        run: |
        install -m 600 -D /dev/null ~/.ssh/id_ed25519
        echo "${{ secrets.PRIVATE_SSH_KEY }}" > ~/.ssh/id_ed25519
        echo "${{ secrets.KNOWN_HOSTS }}" > ~/.ssh/known_hosts
```

### Deploy site

Finally it is time to deploy your site onto your webserver. For this task `scp` is used to copy the content of the built Hugo site in `htdocs` to your HOSTNAME_OR_IP.

For the purpose of this guide, the location of the web directory for the Hugo site will be in `/usr/local/www/hugo-site`. This, ofcourse, can and probably will differ from your installation.

```yaml
    # Deploy site from subfolder htdocs to the webserver 
    - name: Deploy
        run: scp -r htdocs/* github@HOSTNAME_OR_IP:/usr/local/www/hugo-site
```

### Source material

You can find my repository, including the workflow, on [GitHub](https://github.com/TGion/hugo-gion-io). Please feel free to check it out.

## Conclusion

I can now edit the repository of my website with a decent editor (VSCode), push it to GitHub and automatically deploy it to my server. Yeah!

Although the simplicity of my site doesn't really justify the use of CI/CD, it has been a fun experience setting up an automated task to deploy it. The whole experience has also triggered my interest in the fascinating topic of CI/CD. There is so much more to learn.

Besides that, I finally have my website version controlled and easily updated locally and the pushed to GitHub. So thats good too ;)
  
Have fun with your workflows and take care!
