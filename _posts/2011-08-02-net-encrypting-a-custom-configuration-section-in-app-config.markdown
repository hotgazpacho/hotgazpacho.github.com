--- 
layout: post
title: ".NET: Encrypting a custom configuration section in app.config"
excerpt: <p>After reading up a bit on encrypting configuration sections in app.confg, the recommended approach was to create an Installer class that would make the appropriate Configuration API calls to encrypt the specified section. ... What we found out is that since the Installer isn't spinning up an AppDomain for the app you're installing, the assembly containing the custom Configuration classes needs to be in the GAC.</p>
wordpress_id: 148
wordpress_url: http://hotgazpacho.org/2011/08/net-encrypting-a-custom-configuration-section-in-app-config/
date: 2011-08-02 22:56:15 -04:00
---
<h2>The Problem</h2>
<p>We have created a <a href="http://devlicio.us/blogs/derik_whittaker/archive/2006/11/13/app-config-and-custom-configuration-sections.aspx">custom configuration section</a> for our application. The business has requested that this bit of the configuration be encrypted. This encryption must be system-level, not per-user.</p>
<p>After <a href="http://weblogs.asp.net/jgalloway/archive/2008/04/13/encrypting-passwords-in-a-net-app-config-file.aspx">reading</a> <a href="http://www.codeproject.com/KB/security/ProtectedConfigWinApps.aspx">up</a> a bit on encrypting configuration sections in app.confg, the recommended approach was to create an Installer class that would make the appropriate Configuration API calls to encrypt the specified section. Then, have the MSI call this Installer class during the <strong>Install</strong> phase.<br /></p>
<p>The problem with this approach is that it assumes that the configuration section is one of the standard configuration sections provided by the .NET framework. However, due to the way the MSI installation happens, and the way that the stuff in System.Configuration works, the Install method of the Installer class fails with a FileNotFoundException.</p>
<h2>The Solution</h2>
<p>The solution feels like an ugly, nasty hack. Then again, aren't most things related to Configuration and Installation on .NET?</p>
<p>What we found out is that since the Installer isn't spinning up an AppDomain for the app you're installing, the assembly containing the custom Configuration classes needs to be in the GAC. The GAC? Yes, the GAC. I know. Ewwww. We didn't want to GAC the entire application, or even our Core assembly, as they had way to many dependencies that would also need to be GAC'd.</p>
<p>What we ended up doing was extracting the custom Configuration classes to a Core.Config assembly. We then changed the MSI to install this assembly into the GAC. Finally, we changed the Installer class to perform the encryption work during the <strong>Commit</strong> phase. This ensured that the Core.Config assembly had been GAC'd by the time we called <a href="http://msdn.microsoft.com/en-us/library/ms224437.aspx">ConfigurationManager.OpenExeConfiguration(exePath)</a>, the custom Configuration classes would be available for our use.</p>
<p>This was our solution. Please share if you know of a better solution!</p>
