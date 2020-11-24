# Description
The objective of this recipe is to create a deployment pipeline from
scratch and bring our HelloWorld application from coding to production. The project
in question will be deployed in the form of a WAR file on an Apache Tomcat7 server.

This guide should walk you through the installation of a deployment pipeline as
painlessly as possible with mosts steps already automated in various scripts.
The procedure will nonetheless take a good two hours so feel free to grab a cup
of coffee.

Finally, the checkpoint sections are meant to help you ascertain the proper functioning of
your environment. Feel free to skip them if you know what you are doing.

## Conventions
If not specified `$paths` are to be understood as `$GIT_ROOT/$path`
where `$GIT_ROOT` is where you have downloaded or unwrapped this repo.  
`~$ <user-command>`\
`~# <root-command>`


##Requirements before you start
Vagrant 2.2.13 <= \
Ansible 2.10.3 <= (uses Python 2.6) \
Python 3.8 <=

Ansible extra plugin for Docker: `community.general` \
Install with: \
`~:$ ansible-galaxy collection install community.general` \
\[:warning:\] 16GB of RAM is recommended (proceed with less at your own risk)


# 1. Development Environment

## Create development environment 
base install: 3 min with 3000kbps
\+ provisioning: 3 min \
`~$ cd DevEnv/` \
`~$ vagrant up` \
`~$ vagrant ssh`

## Checkpoint: 
`~$ cat /proc/version` \
You should see Ubuntu version 5.4.0 or newer.

## Vagrant Troubleshooting (skip if you have passed the previous checkpoint) 

### I can’t SSH into my machine 
Try deactivating your ssh agent \
Temporarily move `~/.ssh` to `~/.ssh.tmp`  \
Try again
 

### Port collision 
Vagrant normally takes care of collisions. Were it to fail in this regard:
1. Check for open ports on your machine:\
`~# nmap localhost`
2. Adapt the ports in the Vagrantfile accordingly. For example:
```
- 8080:8080
+ 8081:8080
```
3. For more information, visit the Vagrant documentation.

### I’m running out of RAM 
If you have RAM issues, consider closing all other
running windows and operating on only one virtual machine at a time.


After the installation is completed, the source code is accessible through
the inner virtual folder `/vagrant_data/` and outer folder `DevEnv/data/`.  We will
now set up an integration server to have a central repository as well as
automatically build changes we make to the code base.


# 2. Integration Environment

The integration, testing, and production server will all reside on the same
virtual machine but will be logically isolated through the use of docker containers.
We will see how to piece everything together.

## Create integration environment 
We will first create a GitLab CI server that will
also run automated testing.

`~$ cd IntTestProdEnv/` \
`~$ vagrant up` \
`~$ vagrant ssh`

Edit the following file `/etc/gitlab/gitlab.rb` by changing: \
`external_url http://gitlab.example.com` into \
`external ‘http://192.168.33.9/gitlab`

`# unicorn[‘port’] = 8080` into \
`unicorn[‘port’] = 8088`

`~$ gitlab-ctl reconfigure` \
`~$ gitlab-ctl restart unicorn` \
`~$ gitlab-ctl restart`

## Checkpoint: 
Run the following commands: \
`~# gitlab-ctl show-config` \
`~$ docker ps`

You should see respectively the gitlab configuration as well as the tomcat container running.
We will use it later for deployment.

## Using GitLab as a VCS (1/2)
- Navigate to http://192.168.33.9/gitlab with your favorite browser.
- Create an admin password. (To log in with the admin account use (root, `<your-password>`) as
credentials.)
- Register a user account.
- Log in with that user account.
- Create a blank HelloWorld project by following the instructions on GitLab. Keep the project empty for next step.

## Link DevEnv with CI server (2/2)
Armed with this empty GitLab project, go back to your DevEnv: \
`~$ cd DevEnv/` \
`[~$ vagrant up]` \
`~$ vagrant ssh` \
`~$ cd /vagrant_data/`

You will now push the vagrant_data folder that contains the
source code to your GitLab. To do so, go back to your browser and follow the
instructions under “Push and existing folder \[to a remote\]”.  

Your GitLab should now be storing the code directly created in your development 
environment!

**Tip:** For lightning quick access, use the `DevEnv/data/` folder instead of booting up your
 Dev Env.

## Checkpoint: 
Add a blank line to `README.md` and watch the change take place in GitLab.

## CI runner registration 
The last step is to add a runner to automatically build
and test our code.

Let’s return to the IntTestProd environment: \
`~$ cd /IntTestProd/` \
`~$ vagrant ssh` \
`~# sudo gitlab-runner register`

(The command will prompt you the following questions)
Enter the GitLab instance URL \
\> http://192.168.33.9/gitlab/ \
Enter the registration token \
\> \<registration-token\> (can be found in the GitLab project, left pane, Settings, CI / CD; keep track of it for the next step) \
Enter a description for the runner \
\> docker \
Enter tags for the runner \
\> integration \
Enter an executor \
\> docker \
Enter the default Docker image: \
\> alpine:latest

`~$ gitlab-runner start`

Finish off by enabling the runner to accept jobs without tags by visiting the
GitLab browser UI under in your project under runner.

## Checkpoint: 
While on you are on the GitLab repo check if your pipeline is highlighted as green.


# 3. Staging Environment 
Finally, now that we have both our DevEnv and the IntTestEnv, we can complete
the pipeline with our staging environment. The following runner will help by
automatically moving the build artifact to a folder accessible by our tomcat
server.

Staging runner registration  \
`~$ cd /IntTestProd/` \
`~$ vagrant ssh` \
`~# gitlab-runner register` \
(Almost the same as before) \
Enter the GitLab instance URL: \
> http://192.168.33.9/gitlab/
Enter the registration token: \
> <registration-token>
Enter a description for the runner: \
> shell
Enter tags for the runner: \
> integration-shell
Enter an executor: \
> shell
Enter the default Docker image: \
> alpine:latest 

`~# gitlab-runner start`

When you are inside the IntTestProd vagrant server, make sure the gitlab-runner
has access to the `/tomcat_data/` folder: \
`~# mkdir /tomcat_data/` \
`~# chown gitlab-runner /tomcat_data/` \
`~# usermod -aG vagrant gitlab-runner` \

Tomcat is now able to run our war artifacts freshly out of the CI assembly line.

## Checkpoint: 
Open your browser and visit http://192.168.33.9:8888/.  
You should see our Hello-World app displayed!  

Thanks for making it until the end.
