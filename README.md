# How to update a datapackage with travis-ci
This tutorial is about using [travis-ci](http://travis-ci.com) to run packaging scripts to update datapackages. The purpose is to achieve
__continuous processing__ : *deliver updated data every time something changes, either the source data or the processing code*.

The whole tutorial exposes a simplified version of [s-and-p-500-companies](http://data.okfn.org/data/core/s-and-p-500-companies) which scrapes
the list of companies from Wikipedia to creates a datapackage.

Table of Contents
=================

  * [Prerequisite](#prerequisite)
    * [Validation of data](#validation-of-data)
    * [Project must be automated with Travis](#project-must-be-automated-with-travis)
  * [How to update the datapackage from the Makefile ?](#how-to-update-the-datapackage-from-the-makefile-)
    * [Authentication on github](#authentication-on-github)
    * [Work locally](#work-locally)
    * [Push the changes after processing](#push-the-changes-after-processing)
  * [Run the project with travis-ci](#run-the-project-with-travis-ci)
    * [Configure the ssh key](#configure-the-ssh-key)
    * [Configure git](#configure-git)
    * [Publish !](#publish-)
    * [Run the script on the right branch](#run-the-script-on-the-right-branch)
    * [One more step](#one-more-step)
  * [More control over your script](#more-control-over-your-script)
    * [When the source data changes](#when-the-source-data-changes)
    * [But I don't want to spent my time checking if the source data has changed...](#but-i-dont-want-to-spent-my-time-checking-if-the-source-data-has-changed)
  * [How does it affect usage of my tools ?](#how-does-it-affect-usage-of-my-tools-)
    * [What if I create a branch ?](#what-if-i-create-a-branch-)
    * [What if someone forks the project ?](#what-if-someone-forks-the-project-)
    * [What if I restart a former build ?](#what-if-i-restart-a-former-build-)
  * [Discipline](#discipline)
  * [Conclusion](#conclusion)


## Prerequisite
This tutorial assumes you already have a git project that contains a datapackage. This project has a ``script`` 
directory in which the ``Makefile`` retrieve remote data to create the core of the datapackage in the ``data`` directory. 

### Validation of data
Also, you need to have tests to validate the data produced **before** you publish it. See if you don't have tests, stop 
reading this and jump to this really [good article](http://okfnlabs.org/blog/2016/05/17/automated-data-validation.html).

So the [``scripts/Makefile``](scripts/Makefile) in your project should look like our example's :

    all: valid.txt

    List_of_S%26P_500_companies.html:
        curl -o List_of_S%26P_500_companies.html "https://en.wikipedia.org/wiki/List_of_S%26P_500_companies"

    ../data:
        mkdir ../data
        
    ../data/sp500-companies.csv: ../data List_of_S%26P_500_companies.html scrape-wikipedia.py
        python scrape-wikipedia.py

    valid.txt: ../data/sp500-companies.csv ../datapackage.json test_data.py
        python test_data.py
        echo "Datapackage is valid" > valid.txt

    .PHONY: all


Where [scrape-wikipedia.py](scripts/scrape-wikipedia.py) extracts the table from the wikipedia page into a csv file and [test-data.py](scripts/test-data.py)
tests the data.

### Project must be automated with Travis
First, you'll need a [travis](http://travis-ci.org) account !

If you haven't added the project to travis-ci yet, go to your travis profile and switch the project on. Then you can add a ``.travis`` file :

    language: python
    python:
      - 2.7

    install:
      - pip install --upgrade -r scripts/requirements.txt

    script:
      - cd scripts
      - make

Now every time you commit a change in the repository, the data is validated by travis. You'll be notified if the data is invalid.

## How to update the datapackage from the Makefile ?

How to commit file ``sp500-companies.csv`` and push it back to the repository once it has been processed ?

### Authentication on github
To push data to a git repository on github, you need to be authenticated, and the git way is to use ssh keys. So we'll create 
a specific key set for the project :

    ssh-keygen -t rsa -b 4096 -C "Ssh key for repository ex-continuous-processing" -f ~/.ssh/ex-continuous-processing

This command creates two files : 
  * the public key ``ex-continuous-processing.pub`` is the file you can distribute so that servers allow your connections
  * the private key ``ex-continuous-processing`` is the file you need to keep secret on your computer that lets you connect on a server

The easiest way is to be able to push to github is to 
[add the public key to the repository](https://developer.github.com/guides/managing-deploy-keys/#deploy-keys) :
In the repository's *Settings > Deploy Keys*, we'll add a key by pasting the content of ex-continuous-processing.pub. 

![Add public key to github project](pub_key_github.png)

Now we can use the private key to write on the repository.

If we needed to work with several repositories, we could also have created a 
[specific github account](https://developer.github.com/guides/managing-deploy-keys/#machine-users), like
[datasets-update-bot](https://github.com/datasets-update-bot).

### Work locally
    
Before we set travis-ci, we'll configure our local computer to be able to work on the project and test it.

We need to configure our ssh client to use the key we've created with github. Add these lines to your ``~/.ssh/config`` :

    Host github.com
        HostName github.com
        User git
        IdentityFile ~/.ssh/ex-continuous-processing

We should be able to communicate with github :

    $ ssh git@github.com
    > Hi lexman/ex-continuous-processing! You've successfully authenticated, but GitHub does not provide shell access.
    > Connection to github.com closed.

The odds are that your local repository has not been cloned through ssh. That's why we'll add a *remote*
called ``publish`` to our git repository in order to be able to push the changes by ssh.

    git remote add publish git@github.com:lexman/ex-continuous-processing.git

This means that we can share the changes on github with the command ``git push publish``. Let's test it :
    
    touch test-write
    git add test-write
    git commit -m "Testing write from local workstation"
    git push publish

This should work without asking password. Now let's roll-back everything to normal :

    git rm test-write
    git commit -m "Test has succeeded"
    git push publish


### Push the changes after processing
Now that our workstation is able to write to github, we can automate publication with this new rule at the end of the ``Makefile`` :

    pushed.txt: valid.txt
        git add ../data/sp500-companies.csv
        git commit -m "[data][skip ci] automatic update" || exit 0
        git push publish
        echo "Update has been pushed if there was a change" > pushed.txt
    
The first line ``pushed.txt: valid.txt`` means that validation of data must have succeeded before we try publishing. The last 
line creates file ``pushed.txt`` so that running ``make`` again won't run this code again, unless ``valid.txt`` has changed (which means
something has changed in the data).

The git part of this code needs to be explained :
* If there has been no change in ``../data/sp500-companies.csv``, ``git add`` will succeed but won't add any change to the repository. In this case, ``|| exit 0`` in the ``git commit`` line prevents the command from failing... And ``make`` can go on with ``git push`` that will succeed even if there is nothing to publish.
* If there has been some change in file ``../data/sp500-companies.csv``, the code is pretty straightforward... 
    
We'll also change the main rule of the Makefile, because now the purpose of the project is to publish :

    all: pushed.txt

Let's see what it does (it there was no change in the wikipedia page) :

    $ make
    > git add ../data/sp500-companies.csv
    > git commit -m "[data][skip ci] automatic update" || exit 0
    > # On branch master
    > # Your branch is up-to-date with 'origin/master'.
    > #
    > # Untracked files:
    > #	List_of_S%26P_500_companies.html
    > #	valid.txt
    > #
    > nothing added to commit but untracked files present
    > git push publish
    > Warning: Permanently added the RSA host key for IP address '192.30.252.131' to the list of known hosts.
    > Everything up-to-date
    > echo "Update has been pushed if there was a change" > pushed.txt
    
Now we'd need to run from travis rather than our workstation.
    
## Run the project with travis-ci

### Configure the ssh key

At the moment, if we push the code above to github, travis-ci will fail because it has not been configured yet. We'll have to enter
the private key in the project's [settings](https://travis-ci.org/lexman/ex-continuous-processing/settings) so that travis 
can write to github.

![settings in travis](travis_find_settings.png)

Open your private key with your favourite text editor and copy the inner part **without** the header (``-----BEGIN RSA PRIVATE KEY-----``) nor
the footer (``-----END RSA PRIVATE KEY-----``) and paste **into quotes** it into travis settings as an environment variable called ``SSH_PUSH_KEY``. Your 
browser will paste it into one line. Before you add the variable, make sure the *Display value in build log* button is set to off, so that 
your key remains secret all the time.

![set private key in travis](travis_set_key.png)

Now we'll have to recompose the key in the build, by adding a ``before_script`` section in the [.travis.yml](.travis.yml) file :

    before_script:
    # Make the the ssh key available to git
      - echo "-----BEGIN RSA PRIVATE KEY-----" >> ~/.ssh/id_rsa
      - echo "$SSH_PUSH_KEY" | sed 's/ /\n/g' >> ~/.ssh/id_rsa
      - echo "-----END RSA PRIVATE KEY-----" >> ~/.ssh/id_rsa
      - chmod 600 ~/.ssh/id_rsa

### Configure git

We also have to add the ``publish`` remote to the repository in the ``before_script`` section, as we have done in local :

    # Add remote for ssh access
      - git remote add publish git@github.com:$TRAVIS_REPO_SLUG.git


Note that we used the travis ``$TRAVIS_REPO_SLUG`` variable to name the project's repository, so that future forks won't have to 
edit the .travis.yml file.

We also have to set a few more things in git, to avoid useless warning messages and set our identity for commits. The the overall 
``install`` section should look like :

    install:
      - pip install --upgrade -r scripts/requirements.txt
    # Avoid warnings from git :
      - git config --global push.default simple
      - git config --global user.email "you@your-commit-mail.com"
      - git config --global user.name "Your commit bot name"

After all this configuration, Travis will be able to write to our repository...      

### Publish !

Travis has a specific step for deploying a build that is well suited for our task :

    deploy:
      skip_cleanup: true
      provider: script
      script: make pushed.txt
      on:
        all_branches: true

``provider: script`` means we'll run a shell script, and the script is ``make pushed.txt``. The other option ``skip_cleanup: true`` 
is to prevent travis from reverting the changes we made in the build before publish. ``all_branches: true`` should not be difficult 
to understand.


### Run the script on the right branch

Unfortunately this is not enough yet to work, because travis checks out the repository in 
[detached head state](http://alblue.bandlem.com/2011/08/git-tip-of-week-detached-heads.html) so we've got
to get back on the branch we're building to be able to commit. We can fix it in ``script:``, 
but it is easier to add a ``before_deploy`` section :

    before_deploy:
    # Git is in detached HEAD, so we get back on the branch we are supposed to be
      - git checkout -B $TRAVIS_BRANCH 

After we push this last change, we can see our travis build green ! At last !

### One more step

We can even see two green builds ! Because when the first one succeeded, it pushed the data... Which triggered a new build !
We have to add [Skip ci] to the commit message to prevent endless builds. So in the ``Makefile``, the ``git commit`` 
instruction will be changed to :

	git commit -m "[data][skip ci] automatic update" || exit 0

## More control over your script

### When the source data changes

When we know the source data has changed, ie the [wikipedia page about SP500](https://en.wikipedia.org/wiki/List_of_S%26P_500_companies) 
in our example, we don't need to check out the project and run it on our computer any 
more. Now that we have a working travis project, we can just go our [project](https://travis-ci.org/lexman/ex-continuous-processing) and 
click *Restart build* to update the data !

### But I don't want to spent my time checking if the source data has changed...

You can configure your project to run every day, in order to follow updates on the source data. As explained 
in the [travis doc](https://docs.travis-ci.com/user/cron-jobs/), you need to mail the travis support to enable this option.

Also, you'll have to remove the ``[skip ci]`` tag from the commit message, because it would prevent the build from being triggered... So 
you'll have to bear the *double build*.

Now this is the end of your journey : once it's done, your **project auto updates every day and on  every commit** ! Congratulations !


## How does it affect usage of my tools ?

### What if I create a branch ?
Travis will [update the data on the new branch](https://travis-ci.org/lexman/ex-continuous-processing/builds/139785456) ! Thanks to ``on: all_branches: true`` and ``git checkout -B $TRAVIS_BRANCH`` on ``before_deploy``...

### What if someone forks the project ?
Making the data will still succeed, but the pushing part will fail. The owner of the new project will have to configure a new 
set of keys : the public key in the github account, and the private key in the travis settings of the project.

### What if someone submit a pull request ?
Travis will [make the data and will run the tests](https://travis-ci.org/datasets/s-and-p-500-companies/builds/140036140), so you'll know if the data is valid. But hopefully it will [skip the deploy phase](https://docs.travis-ci.com/user/deployment#Pull-Requests).
So you'll know how a a pull request behave, except if the deploy phase was changed.

### What if I restart a former build ?
Restarting a build which is not the last of a branch will fail. But what did you expect ? Only the last commit is allowed 
to publish on a branch, because it would be a mess if you could publish out-of-date data... Pushing will display this error message :

    To git@github.com:lexman/ex-continuous-processing.git
     ! [rejected]        master -> master (non-fast-forward)
    error: failed to push some refs to 'git@github.com:lexman/ex-continuous-processing.git'
    hint: Updates were rejected because the tip of your current branch is behind
    hint: its remote counterpart. Integrate the remote changes (e.g.
    hint: 'git pull ...') before pushing again.
    hint: See the 'Note about fast-forwards' in 'git push --help' for details.
    make: *** [pushed.txt] Error 1

## Discipline

Once *continuous processing* has been achieved, data will update automatically, but it's not effortless.

### Always fix broken windows
Maintaining this system is 
demanding because experience shows builds do break. In our 
example the structure of the wikipedia page is likely to change when people edit it so the [scraping script](scripts/scrape-wikipedia.py) should 
be fixed. Also travis will be updated and default python could migrate to version 3. 

When build breaks, experience with continuous integration shows that it should be repaired quickly or it's never done. Also 
all the value of bringing online new data, is void if existing ones are broken. That's why it's important to be able to work locally 
to solve broken builds.

### Only push if it works
You don't want to share some code or some data that does not work. That's why we configured our project to work locally. You should
always run the project before you push, to detect failures. And avoid the terrible pattern *fix -> push-to-travis -> find-a-new-bug-in-the-logs -> redo* : it's
very time consuming and harmful for your co-workers and people using your data.

### If you find a bug add a test
And remember, before you fix a bug, you should add a test to prevent future regressions !


## Conclusion
With a Makefile, some tests, travis-ci and a bit of maintenance we can achieve **continuous processing** to provide up-to-date quality data :
 anyone can contribute through the well know process of pull-request and validation and publication of datasets will be automatic.
It's very well suited for small data (less then 300 MB) and when the whole processing is short (less 10 minutes), That's why it is 
perfect for core datapackges !

The impatient will implement this tutorial from the shorter [cheat sheet](cheat-sheet.md) !
