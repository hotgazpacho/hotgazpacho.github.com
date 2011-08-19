--- 
layout: post
title: Rails 2.3.0 + IIS7 + FastCGI = Rails on Windows FTW!
wordpress_id: 9
wordpress_url: http://hotgazpacho.org/2009/02/rails-230-iis7-fastcgi-rails-on-windows-ftw/
date: 2009-02-08 01:01:22 -05:00
---
<strong>I love Rails, and I run Windows.</strong>

There, I said it. I know many Railers scoff at us, mock us, etc. Whatever. I run Windows (Vista, specifically). I’m a Microsoft.Net Web Developer at my day job. My company has invested heavily in the Microsoft platform. I don’t like WebForms (why is a topic for another post), and, because of Rails, I know there is a better way to do web development. Yes, I know about <a href="http://www.asp.net/mvc/">ASP.Net MVC</a>, but I have yet to try it. What I have tried is Rails, and I know that a Microsoft stack is the way to get it into my day-to-day work.

So, a couple of days ago, with <a href="http://weblog.rubyonrails.org/2009/2/1/rails-2-3-0-rc1-templates-engines-rack-metal-much-more">the announcement of Rails 2.3.0 rc1</a> I decided to see if I could get the latest Rails running under IIS 7 on my Vista notebook. I tried to find directions on how to accomplish this, but none were very straight forward. So, I’ve decided to document the process here for myself and other WinRailers.

<!--more-->

<em><strong>Note:</strong> 2.3.0 rc1 will not work, due to </em><a href="http://rails.lighthouseapp.com/projects/8994/tickets/1854-fastcgi-dispatcher-dont-work"><em>this bug in Rails' FastCGI handler</em></a><em>. Fortunately, this was fixed and merged into the rails source. You will need the 2.3.0 gem installed, though, to generate a 2.3 Rails app. I’ll go into details below.</em>
<h3>First, Credit where Credit is Due</h3>
I can’t take all the credit for this. The following two articles helped to guide my down my path:
<ul>
	<li><a href="http://mvolo.com/blogs/serverside/archive/2007/02/18/10-steps-to-get-Ruby-on-Rails-running-on-Windows-with-IIS-FastCGI.aspx">10 steps to get Ruby on Rails running on Windows with IIS FastCGI</a></li>
	<li><a href="http://ruslany.net/2008/08/ruby-on-rails-in-iis-70-with-url-rewriter/">Ruby on Rails in IIS 7.0 with URL Rewriter</a></li>
</ul>
<h3>Software You’ll Need</h3>
<ul>
	<li>IIS 7 <em>(Windows Server 2008, or any version of Vista SP1 that has IIS)</em></li>
	<li><a href="http://www.microsoft.com/web/channel/products/WebPlatformInstaller.aspx">The Microsoft Web Platform Installer</a></li>
	<li><a href="http://rubyforge.org/frs/shownotes.php?release_id=17128">Ruby One-Click Installer 1.8.6-26 Final Release</a></li>
	<li><a href="http://code.google.com/p/msysgit/">msysgit</a></li>
	<li><a href="http://sqlite.org/" target="_blank">SQLite3</a></li>
</ul>
<h3>The Steps You’ll Take</h3>
<ol>
	<li>Enable FastCGI support in IIS7</li>
	<li>Install URL Rewrite module</li>
	<li>Install Ruby</li>
	<li>Install msysgit</li>
	<li>“Install” SQLite</li>
	<li>Update Rubygems</li>
	<li>(gem) Install Rails</li>
	<li>Create a Rails app</li>
	<li>Grab the latest Rails 2.3 from github</li>
	<li>web.config</li>
	<li>Setup up web site in IIS</li>
</ol>
<h4>1 - Enable FastCGI support in IIS7</h4>
IIS7 supports FastCGI natively now. You just need to make sure you install CGI support. It was not obvious to me that this was needed. I was looking for a FastCGI option, but it is listed as plain old CGI. <a href="http://learn.iis.net/page.aspx/246/using-fastcgi-to-host-php-applications-on-iis-70/#EnableFastCGI" target="_blank">Microsoft has good directions</a> on how to accomplish this task on their site, so I will not rehash them here.
<h4>2 - Install URL Rewrite Module</h4>
The easiest way to get this (and keep it updated!) is to use the Web Platform Installer. It might also want to install updated FastCGI support. I let this happen with no ill effects. So, go ahead a check “URL Rewrite” in the installer and go to town.
<h4>3 - Install Ruby</h4>
<a href="http://blog.mmediasys.com/" target="_blank">Luis Lavena</a> has done an awesome job of maintaining the Ruby One Click Installer for Windows. Of particular importance to my little endeavor is that, in the <strong>One-Click Ruby Installer 1.8.6-26 Final Release</strong>, Luis included ruby-fcgi and the FastCGI C extension. Thanks, Luis! You can safely run the installer and click “Next” all the way through.
<h4>4 - Install msysgit</h4>
Rails developers use git. You need get to pull the latest source, which includes the FastCGI fix I aluded to above. msysgit is your best option. You can safely run the installer and click “Next” all the way through.
<h4>5 - “Install” SQLite</h4>
I use quotes around install because you don’t really install anything. You simply unzip the sqlite-3_6_10.zip file into any folder in your path. Personally, I create a folder, <code>C:\bin</code>, add that to my path, and unzip the files there. I keep other things here, too, like wget.
<h4>6 - Update Rubygems</h4>
The version packaged in the OCI is old. Save yourself trouble, open up a command prompt, and:
<code>gem up --system</code>
<h4>7 - (gem) install Rails</h4>
You need to have Rails installed. Rails 2.3.0 rc1 has some undeclared dependencies. So, first:
<code>gem install test-spec rack</code>
Next,
<code>gem install rails --source <a href="http://gems.rubyonrails.org">http://gems.rubyonrails.org</a></code>
<h4>8 - Create a Rails app</h4>
Most of my code goes into the following folder: C:\Development. Assuming this directory does not exist:
<ol>
	<li>Open a command prompt</li>
	<li><code>c:</code></li>
	<li><code>cd \</code></li>
	<li><code>mkdir Development</code></li>
	<li><code>cd Development</code></li>
	<li><code>rails myapp</code></li>
</ol>
<h4>9 - Grab the latest Rails from github</h4>
This is where msysgit comes in. Fire up Git Bash, and:
<ol>
	<li><code>cd /c/Development/myapp/vendor</code></li>
	<li><code>git clone git://github.com/rails/rails.git</code></li>
</ol>
<h4>10 – web.config</h4>
The IIS team, under <a href="http://weblogs.asp.net/scottgu/" target="_blank">Scott Guthrie</a> <em>(also oversees ASP.Net, particularly ASP.Net MVC)</em>, have done some awesome stuff in the last couple years. They have turned IIS 7 into, in the words of <a href="http://www.coreyhaines.com/" target="_blank">Software Journeyman Corey Haines</a>, “A respectable web server”. I agree. IIS 7 is a lot more like Apache than it ever used to be, and I think this is a good thing.

One of the nice things is that you can now move a bunch of your web server configuration into a web.config file in your application. That means you can add it to version control :)

ASP.Net programmers will be intimately familiar with this file. Everyone else can think of this file as the equivalent of Apache’s .htaccess files. In our case, we’ll use the following web.config to configure both the FastCGI handler <em>and</em> the URL Rewrite rules:
<pre lang="xml">&lt;?xml version="1.0" encoding="utf-8"?&gt;
&lt;rule name="Rewrite app_offline.html if file exists" stopProcessing="true"&gt;
  &lt;match url="^.*$" ignoreCase="false" /&gt;
  &lt;conditions&gt;
    &lt;add input="{DOCUMENT_ROOT}/app_offline.html" matchType="IsFile" ignoreCase="false" /&gt;
    &lt;add input="{SCRIPT_FILENAME}" pattern="app_offline.html" ignoreCase="false" negate="true" /&gt;
    &lt;add input="{SCRIPT_FILENAME}" pattern="^(.+).(png|gif|jpg|css|js)$" ignoreCase="false" negate="true" /&gt;
  &lt;/conditions&gt;
  &lt;action type="Rewrite" url="/app_offline.html" /&gt;
&lt;/rule&gt;</pre>
So, let me explain what is going on here. First, the handlers section. This is where we set it up so that the FastCGI handler will process requests for dispatch.fcgi with the ruby interpreter, and run myapp in development mode. To run in production, simply replace <em>development</em> with <em>production</em>.

The next section is the URL Rewrite section. As you can see, it is much more, um, <em>expressive</em> than Apache’s syntax. Anyway, another cool thing Microsoft has done with IIS7 and the URL Rewrite module is an Import function. This allows you to import your existing mod_rewrite rules into their format. Cool! So, that’s what I did. I took some existing mod_rewrite rules for FastCGI, and popped it in to the UI to get the markup above.

You’ll notice a section that is commented out. That is an attempt to get the app_offline.html file working. That is, if the file exists, your app is in maintenance mode, and all requests for any Rails route should instead have this served up. I wasn’t able to get this working. Perhaps someone can take a look at it with fresh eyes.
<h4>11 – Set up web site in IIS</h4>
This one is crazy simple. Create a new website, with the physical path set to the public directory of your Rails app. So, in our case, that would be: <code>C:\Development\myapp\public</code>

There is one more thing that I do, but I don’t know if it necessary. For the Application Pool that was created for this site, I change the “.Net Framework Version” to “No Managed Code”. The Rails app is not a .Net app, so I see no reason to turn on managed code for this app.

<a href="http://hotgazpacho.org/wp-content/uploads/2009/02/rails2-3-0iis7apppool.png"><img style="display: inline; border-width: 0px;" title="Rails-2_3_0-IIS7-AppPool" src="http://hotgazpacho.org/wp-content/uploads/2009/02/rails2-3-0iis7apppool-thumb.png" border="0" alt="Rails-2_3_0-IIS7-AppPool" width="644" height="377" /></a>

That's about it! No, seriously. It is that easy. Don’t believe me? Open up a web browser and go to the web site you created (localhost for me). Click on “<a href="http://localhost/rails/info/properties">About your application’s environment</a>”, wait a bit for the FastCGI process to spin up, and voila:

<a href="http://hotgazpacho.org/wp-content/uploads/2009/02/rails2-3-0iis7.png"><img style="display: inline; border-width: 0px;" title="Rails-2_3_0-IIS7" src="http://hotgazpacho.org/wp-content/uploads/2009/02/rails2-3-0iis7-thumb.png" border="0" alt="Rails-2_3_0-IIS7" width="660" height="461" /></a>
