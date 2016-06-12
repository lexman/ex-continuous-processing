# How to update a datapackage with travis-ci
This tutorial is about using [travis-ci](http://travis-ci.com) to run packaging scripts to update datapackages. The purpose is to achieve
__continuous processing__ : *devliver updated data every time something changes : either the source data or the processing code*.

The whole tutorial exposes a simplified version of [s-and-p-500-companies](http://data.okfn.org/data/core/s-and-p-500-companies) which scrapes
the list of companies from Wikipedia to creates a datapackage.



## Why


## Prerequisite
This tutorial assumes you already have a git project that contains a datapackage. This project has a ``script`` 
directory in which the ``Makefile`` retrieve remote data to create the core of the datapackage in the ``data`` directory. Also, 
you need to have tests to validate the data produced **before** you publish it. See if you don't have tests, stop reading this and jump
to this really [good article](http://okfnlabs.org/blog/2016/05/17/automated-data-validation.html).

So the [``scripts/Makefile``](scritps/Makefile) in your project should look like our example's :

    all: ../data/sp500-companies.csv valid.txt

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

## Automate the project with Travis
If you haven't added the project to travis-ci yet, go to your travis profile and switch the project on. Then you can add a ``.travis`` file :

    language: python
    python:
      - 3.4

    install:
      - pip install --upgrade -r scripts/requirements.txt

    script:
      - cd scripts
      - make

Now every time you commit a change in the repository, the data is validatd by travis. You'll be notified if the data is invalid.

## How

The question now is : how to commit file ``sp500-companies.csv`` and push it back to the repository once it has been processed ?

### Authentication on github
To push data to a git repository on github, you need to be authenticated, and the git way is to use ssh keys. We'll create 
a specific key for the project :

    ssh-keygen -t rsa -b 4096 -C "ex-continuous-processing" -f ~/.ssh/ex-continuous-processing

This command creates two files : 
  * the public key ``ex-continuous-processing.pub`` is the file you can distribute to servers to allow your connections
  * the private key ``ex-continuous-processing`` is the file you can keep secret on your computer that lets you connect on a server

The easiest way is to be able to push to github is to 
[add the public key to the repository](https://developer.github.com/guides/managing-deploy-keys/#deploy-keys) :
In the repository's **Settings > Deploy Keys**, let's add a key by pasting the content of ex-continuous-processing.pub. 
SCREENSHOT

Now we can use the private key to write on the repository.

If we needed to work with several repositories, we could also have created a 
[specific github account](https://developer.github.com/guides/managing-deploy-keys/#machine-users), like
[datasets-update-bot](https://github.com/datasets-update-bot).

## Work locally
    
Before we set travis-ci, we'll configure our local computer to be able to work on the project and test it.

We need to configure our ssh client to use the key we've created with github. Add these lines to your ``~/.ssh/config`` :

    Host github.com
        HostName github.com
        User git
        IdentityFile ~/.ssh/ex-continuous-processing

We should be able to communicate with github :
    $ ssh git@github
    > bla bla bla

The odds are that your local repository has not been cloned through ssh. That's why we'll add a **remote**
called ``publish`` to our git repository in order to be able to push the changes by ssh.

    git remote add publish git@github.com:lexman/ex-continuous-processing.git

This means that we can share the changes on github with the command ``git push publish``. Let's test it :
    
    touch test-write
    git add test-write
    git commit -m "Testing write from local workstation"
    git push publish

And roll-back everything to normal :

    git rm test-write
    git commit -m "Test has succeeded"
    git push publish


## Push the changes after processing
Now that our workstation is able to write to github, we can automate publication with this new rule at the end of the ``Makefile`` :

    pushed.txt: valid.txt ../data/sp500-companies.csv
        git add ../data/sp500-companies.csv
        git commit -m "[data] automatic update" || exit 0
        git push publish
        echo "Update has been pushed if there was some change" > pushed.txt
    
The first line ``pushed.txt: valid.txt ../data/sp500-companies.csv`` means that validation of data must have succeeded before we try publishing. The last 
line creates file ``pushed.txt`` so that running ``make`` again won't run this code again, unless ``valid.txt`` has changed.

The git part of this code needs to be explained :
* If there has been no change in ``../data/sp500-companies.csv``, ``git add`` will succeed but won't add any change to the repository. In this case, ``|| exit 0`` in the ``git commit`` line prevents the command from failing... And ``make`` can go on with ``git push`` that will succeed even if there is nothing to publish.
* If there has been some change in file ``../data/sp500-companies.csv``, the code is pretty straightforward... 
    
    $ make pushed.txt
    > blabla
    
# Configure travis-ci to run the project

At the moment, if we push the code above to github, travis-ci will fail because it has not been configured yet. We 
need to modify the ``.travis.yml`` file in order to configure git.

before_script:
  - echo $SSH_PUSH_KEY > ~/.ssh/id_rsa
  - chmod 600 ~/.ssh/id_rsa
  - cat ~/.ssh/config
  - git config --global user.email "datasets-update-bot@lexman.org"
  - git config --global user.name "Update bot"


  
[Skip ci]


# IDEAS

	git commit -m "[data] automatic update" || echo "No change" > pushed.txt ; exit 0
	git push
	echo "Update has been pushed if there was a change" > pushed.txt
    
    
Tags