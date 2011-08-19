--- 
layout: post
title: "IronRuby: `gem update --system`"
wordpress_id: 106
wordpress_url: http://hotgazpacho.org/?p=106
date: 2010-05-18 12:28:19 -04:00
---
<p>After reading <a href="http://marcinobel.com/index.php/bug-invalid-exec_format-ir" target="_blank">this article</a> on getting around the <strong><em>invalid exec_format “ir”, no %s</em></strong> issue when trying to update RubyGems in IronRuby, I came up with a better solution (rather than essentially swallowing the exception) to the problem. Stick the following file in:<tt> %IronRubyInstallDir%\lib\ruby\site_ruby\1.8\rubygems\defaults\ironruby.rb</tt></p> [ccW_ruby tab_size="2"]
module Gem
  def self.default_exec_format
    exec_format = ConfigMap[:ruby_install_name].sub('ir', '%s') rescue '%s'

    unless exec_format =~ /%s/ then
      raise Gem::Exception,
        "[BUG] invalid exec_format #{exec_format.inspect}, no %s"
    end

    exec_format
  end
end
[/ccW_ruby]
<p>I'm working on getting this contributed to the IronRuby project, as judging from the RubyGems sources, as well as the RubyGems sources distributed with JRuby, this is intended to be distributed by the Ruby implementation, not RubyGems itself.</p>
<p>Also note, this allows one to:</p>
<code>ir -S gem install bundler</code>
