---
layout: post
title:  "How to develop SharePoint Features on a remote server"
date:   2006-10-23
---


<h2>
Why I use remote server?</h2>
Usually all SPS 2007 developers create their programs on localhost server and later they publish them onto production one.   But I have not that much RAM to install server onto my developer machine. Plus - using remote machine as a server is pretty convenient, cause you may test your programme in a battle field, which should help you to avoid different code security problems.     The choice is up to you, but even on local machine some parts of my story will be usable.   
<h2>
What is the remote server developing outline?</h2>
<ol>
<li>Writing/modifying your programme</li>
<li>Build the project</li>
<li>Starting deploy procedure</li>
<li>See the results in browser</li>
<li>Optionally: use Remote Debugging</li>
</ol>
Hmm... and here are some comments on these steps:  
<h2>
Is it really possible to create SharePoint programs without a server on the localhost?</h2>
Yes, of course. But you'll need some brain work to make it working correctly.    The process of developing is straightforward, you create source files, then compile them in dll, then copy them to server. And, hop!, the server is using your new Feature.    The main problem here is to build dll files from sources. You'll need many *.dll files from SharePoint Server (or WSS, if you are not using Server functionality). They can be found on your hard drive (i never could remember the correct paths). Their names are like Microsoft.SharePoint.dll (the only lib i'm currently using).    Also you'll need *.xsd files for Feature.xml intellisense developing. You can copy them from one of the *.cab files of SharePoint distribution (or from the installed version of one).  Their names are: CamlQuery.xsd camlview.xsd coredefinitions.xsd wss.xsd wss12.xsd    Tip: if you extract them from *.cab file, then you would need some renaming, like camlqry.xsd to CamlQuery.xsd    
<h2>
How to deploy your Feature onto the server?</h2>
First. You should make post-build event, to make it possible to redeploy Feature on the server by just one click of Ctrl-Shift-B. (see <a href="http://ripos.blogspot.com/2006/10/tip-for-creating-post-build-event-in.html">http://ripos.blogspot.com/2006/10/tip-for-creating-post-build-event-in.html</a>)    <a href="http://photos1.blogger.com/blogger2/6286/1008001056893888/1600/feature-project-structure.png" onblur="try {parent.deselectBloggerImageGracefully();} catch(e) {}"><img alt="" border="0" src="http://photos1.blogger.com/blogger2/6286/1008001056893888/320/feature-project-structure.png" style="cursor: hand; cursor: pointer; float: right; margin: 0 0 10px 10px;" /></a>Personally, I'm using the next Directory hierarchy in the project:    All files that were created by me are in Feature.Xxx namespace and SolutionDir/ProhectName/Feature/Xxx directory. All the libraries, that I use (*.dll SharePoint assemblies), deploy sript and some utils are all in the SolutionDir/lib directory. This directory structure will be used in my deploy script. If you want another structure, just rewrite the part of the script.    
<h2>
What do we need to do in the deploy script?</h2>
<ol>
<li>Delete old files from server</li>
<li>Copy Feature.xml and other Feature-specific files to Features/Xxx directory on the server</li>
<li>Copy our assembly code to server</li>
<li>Register our assembly in GAC</li>
<li>Copy *.pdb *.config files to assembly's GAC dir</li>
<li>Restart IIS</li>
<li>Reinstall Feature</li>
</ol>
<h4>
What utils do I use?</h4>
Of course I'm using some utils to help me make the deploy process easier.    I'm using PsExec from sysinternals.com to remotely run gacutils and iisrestart and stsadm. This util can run programs on the remote machine and impersonalize if you'll need it.    
<h4>
What .bat script I'm using.</h4>
<pre class="brush: bash">@echo off
REM # Installation of the Feature to the MOSS2007 server
REM # Modified on 2006-10-14
REM # Created by ripos: timofey.basanov@gmail.com http://timofey.basanov.googlepages.com
REM ____________________________________________
REM Setting Server location:
SET Server=portal
SET ServerDrive=C

REM Files needed to copy to \bin (including .config, .pdb)
SET FeatureAssembly=XxxFeatures.dll
SET FeatureAssemblyFiles=XxxFeatures.*
SET FeatureDirectory=Xxx
SET FeatureAssemblyName=XxxFeatures
SET FeatureAssemblyVersion=1.0.0.0
SET FeatureAssemblyToken=3dbcc3aea4786504

REM If there are not enoegh options start with default ones
if "%4"=="" echo I  Starting with default params
if "%4"=="" "C:\ripos\Develop\VS2005\Sps2007\lib\deployListUserAccess.bat" ListUserAccess C:\ripos\Develop\VS2005\Sps2007\ListUserAccess\ C:\ripos\Develop\VS2005\Sps2007\ C:\ripos\Develop\VS2005\Sps2007\ListUserAccess\bin\Debug\

REM Loading settings from command line parameters
echo .. Loading project settings
SET ProjectName=%1
SET ProjectDir=%2
SET SolutionDir=%3
SET OutDir=%4
SET LibDir=%3lib
echo    Build parameters: project %1,
echo                      located in %2,
echo                      solution located in %3,
echo                      builded to %4,
echo                      libraries located in %LibDir%

REM Creating root paths:
SET ServerRoot=\\%Server%\%ServerDrive%$
SET LocalServerRoot=%ServerDrive%:
echo    Using server: %ServerRoot%

REM Setting local paths
SET SharePointFolder=Program files\Common files\Microsoft shared\Web server extensions\12
SET FeaturesPath=%SharePointFolder%\TEMPLATE\Features
SET IISSite=Inetpub\wwwroot\wss\VirtualDIrectories\80
SET GAC=WINDOWS\assembly\GAC_MSIL

REM Settings source/deploy paths
SET FeatureSourcePath=%OutDir%\Features\%FeatureDirectory%
SET FeatureDeployPath=%ServerRoot%\%FeaturesPath%\%FeatureDirectory%
SET FeatureAssemblyDeployPath=%ServerRoot%\%IISSite%\bin
SET FeatureAssemblyDeployLocalPath=%LocalServerRoot%\%IISSite%\bin
SET FeatureGACPath=%ServerRoot%\%GAC%\%FeatureAssemblyName%\%FeatureAssemblyVersion%__%FeatureAssemblyToken%

echo    Deploying from: %FeatureSourcePath%
echo        (xml) to:   %FeatureDeployPath%
echo        (dll) to:   %FeatureAssemblyDeployPath%
SET Remote="%LibDir%\psexec.exe" \\%Server%
rem  -u %RemoteUser% -p %RemoteUserPassword%
rem We are using -d switch to workaround bug in VS2005 that hangs on on unknown reason
SET StsAdm=%Remote% -d "%LocalRoot%\%SharePointFolder%\BIN\stsadm.exe"
SET GacUtil=%Remote% -c "%LibDir%\gacutil.exe" /nologo
SET IISReset=%Remote% "iisreset"

echo .. Starting build of %ProjectName%

echo .. Cleaning old files from the server
md "%FeatureDeployPath%"
del /q "%FeatureDeployPath%\*"
if %errorlevel% NEQ 0 goto Error
del /q "%FeatureAssemblyDeployPath%\%FeatureAssemblyFiles%"
if %errorlevel% NEQ 0 goto Error

echo .. Copying feature xml files to server
copy "%FeatureSourcePath%\*" "%FeatureDeployPath%\"
if %errorlevel% NEQ 0 goto Error

echo .. Copying feature assembly file to server
copy "%OutDir%\%FeatureAssembly%" "%FeatureAssemblyDeployPath%\"
if %errorlevel% NEQ 0 goto Error

rem 1) We do not need no more. We deply directly in \bin
rem 2) We need this, non GAC assemblies are not allowded
echo .. Registering assemblies in GAC
%GacUtil% /i "%FeatureAssemblyDeployLocalPath%\%FeatureAssembly%" /f
if %errorlevel% NEQ 0 goto Error

echo .. Copying feature assembly config and debug files to server
copy /Y "%OutDir%\%FeatureAssemblyFiles%" "%FeatureGACPath%\"
if %errorlevel% NEQ 0 goto Error


REM It's not really needed, but after killing w3wp we make sure, that we do not
REM start dubugging of incorrect version.
echo .. Restarting IIS server
%IISReset% &gt;nul
if %errorlevel% NEQ 0 goto Error

echo .. Reinstalling %FeatureName% Feature
%StsAdm% -o installfeature -name %FeatureName% -force
REM if %errorlevel% NEQ 0 goto Error
REM We run StsAdm with -d switch for PsExec, so resultcode = process ID
REM Success = ProcessID is in [500 .. 50 000]
if errorlevel 50000 goto Error
if errorlevel 500 goto Success
goto Error

REM # The build looks like it succeeded.
goto Success

:Error
echo !! Build failed
REM If errorlevel is not 0 then return it, else return 1
if %errorlevel% NEQ 0 exit
exit 1

:Success
echo I  Build successful
exit 0
</pre>
The only problem with this script is that it sometimes hangs the VS. Then you should kill psexec process from TaskManager.    The other problem is that you'll need to understand this script and change the variables at the beginning to your needs. This will require brains. This is not simple.    
<h2>
How do I remotely debug my program?</h2>
The key idea is simple - to lay *.pdb files in the same directory as *.dll files. Then it'll be possible to debug programme remotely. If you have placed your files in GAC (and you'll need it for EventListener) then you would need to place *.pdb in... in some really non-readable-name directory like \windows\assmbly\GAC_MSIL\XxxFeature\1.0.0.0__122335987743261\  May be you'll need local admin account on server for your developer account.    All that you'll need from VS is to press Debug/AttachToProcess/Portal/w3wp.exe (select all instances).    
<h2>
And... what are the results?</h2>
The results are just super!    I modify the programme, press Ctrl-Shift-B, resfresh in IE (to wake up IIS), AttachToProcess and can debug the progamme not using any resources on my local computer.    Even more, I can deploy it on any other server in the network, by changing only one parameter in the script. Hooray!