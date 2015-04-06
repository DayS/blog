---
layout: post
title:  "Migrate self-hosted repo to Bitbucket"
date:   2015-04-03 12:00:00
categories: bash
tags: bash git bitbucket
header-img: assets/article_images/server-cover1.jpg
---

* https://designhammer.com/blog/easily-migrate-git-repositories-bitbucket
* http://www.cyberciti.biz/faq/linux-script-to-prompt-for-password/

Update script to handle my needs
Do not use hard-coded ids
Use new API


```
#!/bin/bash

# Usage: ./bitbucket-migrate.sh
# Should be launched from the directory containing all Git repos

read -p "Bitbucket team: " TEAM
read -p "Bitbucket username: " USERNAME
read -s -p "Bitbucket password: " PASSWORD

LOG_FILE="migration-script.log"

echo ""

function log {
    echo "$1"
    echo "$(date) > $1" >> "$LOG_FILE"
}

for REPO in *; do
	log "###"

    log "Processing $REPO"
    cd $REPO
    
    log "Creating repo in Bitbucket"
    CREATE_RESULT=`curl --silent -X POST --user $USERNAME:$PASSWORD --header "Content-Type: application/json" --data '{"scm": "git", "is_private": "true", "fork_policy": "no_public_forks"}' "https://api.bitbucket.org/2.0/repositories/$TEAM/$REPO"`
    log "$CREATE_RESULT"
    
    log "Pushing mirror to bitbucket"
    git push --mirror git@bitbucket.org:$TEAM/$REPO
    
    cd ..
    log "Waiting 1 second"
    sleep 1;
done
```