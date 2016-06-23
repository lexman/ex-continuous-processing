# Continuous processing with Travis-ci cheat sheet

you already have read the [tutorial](README.md) and you want to go the fast way to apply it to your project.


## Automate from travis
Enable building your project from your travis account, then add the following ``.travis.yml`` (and 
don't forget to replace the identity of the committer) :

    language: python
    python:
      - 2.7

    install:
      - pip install --upgrade -r scripts/requirements.txt
    # Avoid warnings from git :
      - git config --global push.default simple
      - git config --global user.email "datasets-update-bot@lexman.org"
      - git config --global user.name "Update bot"

    before_script:
    # Add the ssh key
      - echo "-----BEGIN RSA PRIVATE KEY-----" >> ~/.ssh/id_rsa
      - echo "$SSH_PUSH_KEY" | sed 's/ /\n/g' >> ~/.ssh/id_rsa
      - echo "-----END RSA PRIVATE KEY-----" >> ~/.ssh/id_rsa
      - chmod 600 ~/.ssh/id_rsa
    # Add remote for ssh access
      - git remote add publish git@github.com:$TRAVIS_REPO_SLUG.git

    script:
      - cd scripts
      - make valid.txt

    before_deploy:
    # Git is in detached HEAD, so we get back on the branch we are supposed to be
      - git checkout -B $TRAVIS_BRANCH

    deploy:
      skip_cleanup: true
      provider: script
      script: make pushed.txt

## Set the keys
You should have a pair of public / private ssh keys.

In the settings of your github project, paste the public key :

In the settings of your travis project, paste the **inner part** of your private key (without header nor foter) **into quotes** to create
a ``SSH_PUSH_KEY`` variable.

You can also set the private key in your git config to work localy  :
    Host github.com
        HostName github.com
        User git
        IdentityFile ~/.ssh/my-ssh-private-key


## Add publish step to the build
Make sure the tests are run after the data is build, and add the publish step. The end of your Makefile should look like this :

    valid.txt: ../data/MY_PRODUCED_FILE.csv ../datapackage.json test_data.py
        python test_data.py
        echo "Datapackage is valid" > valid.txt

    pushed.txt: valid.txt
        git add ../data/MY_PRODUCED_FILE.csv
        git commit -m "[data][skip ci] automatic update" || exit 0
        git push publish
        echo "Update has been pushed if there was a change" > pushed.txt


# Enjoy !