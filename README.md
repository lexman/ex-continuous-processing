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
If you haven't added the project to travis-ci yet, go to your travis profile and switch the project on. Now you can add a ``.travis`` file :

    language: python
    python:
      - 3.4

    install:
      - pip install --upgrade -r scripts/requirements.txt

    script:
      - make



## How

The question now is : how to commit file ``s-p-500-companies.csv`` and push it back to the repository once it has been processed ?

### Authentication
You can push data to a git repository on github

# 
