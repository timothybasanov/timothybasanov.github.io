---
layout: post
title:  "TeamCity + Gerrit integration for verified patchset statuses"
date:   2011-08-31
---


Recently I've made it possible to integrate TeamCity and Gerrit code review tool. As far as I know there is no page in the internets about this particular integration, then here is a post.

<!--more-->

First of all I would like to thank two people, that have helped me a lot. First is Chris Smith who as far as I know have done it first in this ticket: <a href="http://jira.dmdirc.com/browse/INFRA-25">http://jira.dmdirc.com/browse/INFRA-25</a>, I've got an inspiration and direct help from him. Second is brilliant Unix systems administrator Andrey Domas, who helped me with the Unix side.

# How does it work from the developer side?

  - You upload your patch-set as usual to Gerrit
  - After several seconds there is either Fails or Verified status set for this patch-set, which is done by TeamCity after running unit-tests.

That's it! There is absolutely no additional burden for developer.

# How does it work from overall perspective?

  - There is `patchset-created` hook in Gerrit, which is simple script that runs wget to call TeamCity ([documentation](http://gerrit.googlecode.com/svn/documentation/2.1.2/config-hooks.html)) and TeamCity starts specially configured build in response ([documentation](http://confluence.jetbrains.net/display/TCD5/Accessing+Server+by+HTTP))
  - This build fetches and checkouts change for this patch-set
  - After that it runs through fast and simple unit-tests
  - In case of success or failure there is call to Gerrit through ssh to change patch-set verification status ([documentation](http://gerrit.googlecode.com/svn/documentation/2.0/cmd-approve.html))

So, basically you'll need to write several-lines-long script and configure another one build in TeamCity. And that's all!

# And, how do I do all these things in detail, you ask?

First, I would like to notice that I could not show real production scripts so I'm writing all of them directly from my head, thou they are unverified, untested and have never been run. Beware and test your scripts thoroughly.

To manage user right you will need user in TeamCity who will be the representative of Gerrit Hooks and one user in Gerrit who will be representative of TeamCity Verificator. 

Gerrit user in TeamCity should be able to run our particular build and TeamCity user in Gerrit should be able to place Verified or Fails statuses in specified project. 

I, personally, saved Gerrit login and password for TeamCity directly in the script file, and placed right keys on every build machine to authenticate to Gerrit from TeamCity build agents.

  - Gerrit hook `patchset-created` (at least you should replace project id, build id, login and password with your data): 

    ```python
#!/usr/bin/env python
import sys,getopt,subprocess
opts, args = getopt.getopt(sys.argv[1:], '', ['change=','change-url=','project=','branch=','uploader=','commit=','patchset='])
dic = dict(opts)
if dic['--project']=="fooProject":
    subprocess.call(['wget','--user','gerrit','--password','p@ssw0rd',"http://teamcity.server/httpAuth/action.html?add2Queue=bt226&amp;system.name=system.PATCHSET&amp;system.value=" + dic['--commit']])
```
  - Fetch and checkout for git is a bit tricky, as you will need to:
    - First make sure that build agent could have it's local git repo thus you should select '<i>Automatically on agent</i>' checkout mode in build VCS configuration
    - Secondly make sure that specific refs/changes/* branch is fetched in local repository thus do 
    
      ```sh 
      git fetch origin +refs/changes/*:refs/remotes/origin/changes/*
      ```
    - Lastly you should checkout right commit to run tests in 

      ```sh
      git checkout %system.PATCHSET%
      ```
  - For running tests I'm using Maven, so this is as simple as 

    ```sh
    mvn test
    ```
  - For updating tests I'm using simple ssh just like this: 
  
    ```sh
    ssh -o StrictHostKeyChecking=no gerrit.server -p 29418 -t "gerrit approve --verified=+1 %system.PATCHSET%"
    ```

That's the main line for these scripts. Modify and use them as you want.

I hope that there will be even more happy users of continuous integration systems including you.
