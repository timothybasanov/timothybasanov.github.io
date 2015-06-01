---
layout: post
title:  "TeamCity + Gerrit integration for verified patchset statuses"
date:   2011-08-31
---


Recently I've made it possible to integrate TeamCity and Gerrit code review tool. As far as I know there is no page in the internets about this particular integration, then here is a post.
<div>
</div>
<div>
First of all I would like to thank two people, that have helped me a lot. First is Chris Smith who as far as I know have done it first in this ticket: <a href="http://jira.dmdirc.com/browse/INFRA-25">http://jira.dmdirc.com/browse/INFRA-25</a>, I've got an inspiration and direct help from him. Second is brilliant Unix systems administrator Andrey Domas, who helped me with the Unix side.</div>
<div>
</div>
<h2>
How does it work from the developer side?</h2>
<div>
<ol>
<li>You upload your patch-set as usual to Gerrit</li>
<li>After several seconds there is either Fails or Verified status set for this patch-set, which is done by TeamCity after running unit-tests.</li>
</ol>
</div>
<div>
That's it! There is absolutely no additional burden for developer.</div>
<div>
</div>
<h2>
How does it work from overall perspective?</h2>
<div>
<ol>
<li>There is <span class="Apple-style-span">patchset-created</span> hook in Gerrit, which is simple script that runs wget to call TeamCity (<a href="http://gerrit.googlecode.com/svn/documentation/2.1.2/config-hooks.html">documentation</a>) and TeamCity starts specially configured build in response (<a href="http://confluence.jetbrains.net/display/TCD5/Accessing+Server+by+HTTP">documentation</a>)</li>
<li>This build fetches and checkouts change for this patch-set</li>
<li>After that it runs through fast and simple unit-tests</li>
<li>In case of success or failure there is call to Gerrit through ssh to change patch-set verification status (<a href="http://gerrit.googlecode.com/svn/documentation/2.0/cmd-approve.html">documentation</a>)</li>
</ol>
<div>
So, basically you'll need to write several-lines-long script and configure another one build in TeamCity. And that's all!</div>
</div>
<div>
</div>
<h2>
And, how do I do all these things in detail, you ask?</h2>
<div>
</div>
<div>
First, I would like to notice that I could not show real production scripts so I'm writing all of them directly from my head, thou they are unverified, untested and have never been run. Beware and test your scripts thoroughly.</div>
<div>
</div>
<div>
To manage user right you will need user in TeamCity who will be the representative of Gerrit Hooks and one user in Gerrit who will be representative of TeamCity Verificator. </div>
<div>
Gerrit user in TeamCity should be able to run our particular build and TeamCity user in Gerrit should be able to place Verified or Fails statuses in specified project. </div>
<div>
I, personally, saved Gerrit login and password for TeamCity directly in the script file, and placed right keys on every build machine to authenticate to Gerrit from TeamCity build agents.</div>
<div>
</div>
<div>
<ol>
<li>Gerrit hook <span class="Apple-style-span">patchset-created</span> (at least you should replace project id, build id, login and password with your data): <pre class="brush: python">#!/usr/bin/env python
import sys,getopt,subprocess
opts, args = getopt.getopt(sys.argv[1:], '', ['change=','change-url=','project=','branch=','uploader=','commit=','patchset='])
dic = dict(opts)
if dic['--project']=="fooProject":
    subprocess.call(['wget','--user','gerrit','--password','p@ssw0rd',"http://teamcity.server/httpAuth/action.html?add2Queue=bt226&amp;system.name=system.PATCHSET&amp;system.value=" + dic['--commit']])</pre>
</li>
<li>Fetch and checkout for git is a bit tricky, as you will need to:<ol>
<li>First make sure that build agent could have it's local git repo thus you should select '<i>Automatically on agent</i>' checkout mode in build VCS configuration</li>
<li>Secondly make sure that specific refs/changes/* branch is fetched in local repository thus do <pre class="brush: bash">git fetch origin +refs/changes/*:refs/remotes/origin/changes/*</pre>
</li>
<li>Lastly you should checkout right commit to run tests in <pre class="brush: bash">git checkout %system.PATCHSET%</pre>
</li>
</ol>
</li>
<li>For running tests I'm using Maven, so this is as simple as 'mvn test'</li>
<li>For updating tests I'm using simple ssh just like this: <pre class="brush: bash">ssh -o StrictHostKeyChecking=no gerrit.server -p 29418 -t "gerrit approve --verified=+1 %system.PATCHSET%"</pre>
</li>
</ol>
<div>
That's the main line for these scripts. Modify and use them as you want.</div>
</div>
<div>
</div>
<div>
I hope that there will be even more happy users of continuous integration systems including you.</div>
