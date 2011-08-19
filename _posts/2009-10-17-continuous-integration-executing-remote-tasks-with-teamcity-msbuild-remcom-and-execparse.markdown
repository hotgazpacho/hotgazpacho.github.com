--- 
layout: post
title: "Continuous Integration: Executing Remote Tasks with TeamCity, MSBuild, RemCom, and ExecParse"
wordpress_id: 70
wordpress_url: http://hotgazpacho.org/2009/10/continuous-integration-executing-remote-tasks-with-teamcity-msbuild-remcom-and-execparse/
date: 2009-10-17 23:48:09 -04:00
---
I spent most of last week at work improving our build process, so I thought I’d share with you, and my future self, how I accomplished this.
<h3>First, Why?</h3>
We practice continuous integration (CI) at my office. This means we have a server, specifically <a href="http://www.jetbrains.com/teamcity/">TeamCity</a>, monitor our source control repository and perform various builds on a clean system (thus avoiding the “it works on my machine!” syndrome). In addition to performing continuous builds (whenever someone checks code in), we also perform full builds at regular intervals throughout the day. We also perform a more exhaustive nightly build, where in addition to compiling our suite, we also deploy the built suite to our lab server for more processing. This additional processing entails running a program to create database schemas on the 3 database platform we support (SQL Server, Oracle, &amp; Access), perform conversions from one version of the suite to another, and validate the results.

Sounds straight-forward enough, right? Well, there’s one really big wrinkle here: our build servers are physical servers located in Texas, our lab servers (app server and two database servers) are virtual machines located in Florida, and between them is, shall I say, a <em>really crappy</em> network connection. How crappy? Well, if we run the database creation and validation app on our lab app server, the process takes about 45 minutes. If we run the creation and validation app on our build servers, the process takes <em>significantly</em> longer (I’ve been told 4-6 hours). That extended length is not acceptable, as it interferes with our ability to create the MSIs that we use the next day for integration testing and QA.

To get around this, our nightly build process stopped at copying a zip archive of the compiled suite to our lab server. We then relied on a couple of scheduled tasks on the lab server to run batch files to unzip the suite and run the database creation and validation app. This worked, but had a major draw back. If the creation and validation script failed, it sent an email. However, only a very small number of the team received this email. Since the CI server had no knowledge of this process, it could not fail the build, and the majority of the team would go on not knowing that the suite was effectively broken in a way that would only be apparent at run-time, and only if we happened to exercise a certain bit of functionality that relied on the database schema being valid.

In other words, the risk of massive wasted team productivity was very high.

<!--more-->
<h3>Now, How</h3>
As I mentioned before, we use TeamCity as our CI server. We build a suite of applications for Windows clients and servers, so we use MSBuild as our build system. MSBuild has a built-in task called Exec, which allows you to execute any command, and if the command returns a non-zero exit code, it is considered to have failed. This is great, except that it executes the command on the current machine (recall that the build server is in Texas, but we need the database app to run on the server in Florida.
<h4>PsExec: A False Start</h4>
My first thought for remotely executing commands on a Windows server was to use <a href="http://technet.microsoft.com/en-us/sysinternals/bb897553.aspx">PsExec</a> from the <a href="http://technet.microsoft.com/en-us/sysinternals/default.aspx">SysInternals</a> Suite. This worked great, as long as I ran the MSBuild script from my local machine. However, as I discovered after combing the SysInternals forums, PsExec does some funky things with standard in (stdin) and standard error (stderr), such that Java processes (which TeamCity is), seem to hang when calling PsExec. That is, PsExec does not appear to return control to the spawning Java process in the manner that it expects. So, even though the remote processes would execute properly, they would not return control to TeamCity, nor would it provide the exit code of the remote process. As a result, not only would the build fail (we configured TeamCity to fail the build if it ran more than 2 hours), but it would fail for the wrong reason, and continue to mask the real reason from the majority of the team.

This would not do, no sir. So I set out in search of an alternative.
<h4>RemCom: A New Hope</h4>
After much searching, I came across <a href="http://talhatariq.wordpress.com/projects/remote-command-executor-xrce/">RemCom</a>, “the open source psexec”. RemCom performed the same function as PsExec, but it did so without funky stdin/stderr redirection. So, I added RemCom to the build process, adjusted the build scripts accordingly, and kicked off a new nightly build in TeamCity. The build finished in about 90 minutes, and it succeeded! Except for one problem: I knew there was still a problem with the databases, so I was expecting it to fail. Sure enough, I checked the logs and found that RemCom had provided all the output of the remote commands (something PsExec never did, btw), including printing the exit code returned. However, it seems that exit code was not being bubbled up to RemCom. So, we had a situation where RemCom was telling us the remote command failed, but RemCom itself was returning a successful exit code, and so MSBuild saw this as successful.
<h4>Exec Task: The Empire Strikes Back</h4>
It was great that I was getting the output of the remote commands, but now I needed to parse it. Unfortunately, the Exec task that comes with MSBuild simply does not provide you with the ability to capture the output and do anything meaningful with it. So, it was time to look elsewhere.
<h4>ExecParse: The Return of the Exit Code</h4>
After a little bit of Googling, I came across <a href="http://code.google.com/p/execparse/">ExecParse</a>:
<blockquote>A custom MSBuild task that inherits from the Exec MSBuild task. The task adds a parameter to allow parsing of the output using regular expressions, and reporting to the MSBuild logger (and consequently the VS.Net IDE).</blockquote>
Bingo!

I then added a reference to the task library in our build script and followed the simple directions to create a configuration with a simple regular expression to grab the exit code of the remote command from RemCom. 

[ccie_xml tab_size="2"]
&lt;UsingTask AssemblyFile=&quot;ExecParse.dll&quot; TaskName=&quot;ExecParse.ExecParse&quot; /&gt;
&lt;PropertyGroup&gt;
&lt;DeployServerHost&gt;nasrvfl7004&lt;/DeployServerHost&gt;
&lt;DeployServerLocalPath /&gt;
	&lt;ExecParseConfiguration&gt;
		&lt;Error&gt;
			&lt;Search&gt;Remote command returned (-?[1-9]\d*)&lt;/Search&gt;
			&lt;Message&gt;Remote command failed with error code $1&lt;/Message&gt;
		&lt;/Error&gt;
	&lt;/ExecParseConfiguration&gt;
&lt;/PropertyGroup&gt;

&lt;Target Name=&quot;CreateDbsOnServer&quot;&gt;
&lt;Message Text=&quot;Creating databases on Server $(DeployServerHost)...&quot; /&gt;
&lt;Error Text=&quot;Build and Deploy selected, but the DeployServerHost was not not specified&quot; Condition=&quot;'$(DeployServerHost)'==''&quot;&gt;&lt;/Error&gt;
&lt;Error Text=&quot;Build and Deploy selected, but the DeployServerLocalPath was not specified&quot; Condition=&quot;'$(DeployServerLocalPath)'==''&quot;&gt;&lt;/Error&gt;
	&lt;Copy SourceFiles=&quot;RemCom.exe;CreateDbsOnServer.bat&quot; DestinationFolder=&quot;$(DeployLocation)&quot; /&gt;
	&lt;ExecParse
			Command=&quot;RemCom.exe \\$(DeployServerHost) /d:$(DeployServerLocalPath) $(DeployServerLocalPath)\CreateDbsOnServer.bat $(DeployServerLocalPath)&quot;
			Configuration=&quot;$(ExecParseConfiguration)&quot; 
			ErrorCausesFail=&quot;true&quot; /&gt;
&lt;/Target&gt;
[/ccie_xml]

That, right there, was the secret sauce. Now, we can execute remote tasks from our CI build, include their output in the build log, and fail the build if those commands fail. NICE!

Going forward, we’re going to investigate including the remote commands into the full builds, as they run reasonably quickly enough. We will probably want to investigate adding another build agent, as this would tie up the build server for 90 minutes, thus preventing the continuous builds from running in a timely manner.

<h3>Epilogue</h3>
Pavel Sher, presumably from JetBrains, contacted me via Twitter and <a href="http://twitter.com/spav5/status/4917299743">suggested that I try the latest TeamCity EAP</a>. Pavel tells me that PsExec is working now with this version (would be a 5.0 EAP). Though I am not in a position to go upgrading TeamCity right now, I thought I should include this bit of info, in case someone reading this is.
