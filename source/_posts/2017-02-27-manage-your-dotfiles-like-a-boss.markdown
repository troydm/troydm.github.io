---
layout: post
title: "Manage Your dotFiles like a Boss"
date: 2017-02-27 16:18:50 +0400
comments: true
categories: [ruby,dotfiles,git]
---

TL;DR Writing your own Ruby DSL language to sync and manage your public and private dotfiles

As number of computers I own increased over the years and I couldn't bring myself to get rid of some of the older ones so I thought that it was time to step down and think about management of dotfiles in more centralized, intuitive and automated way, because old ways of just [git](https://git-scm.com/) cloning weren't enough anymore.
Now there are systems that are actually tailored over that particular task such as [GNU Stow](https://www.gnu.org/software/stow/) and [some other tools](https://dotfiles.github.io/) however being a fairly lazy person I thought that writing my own would be a lot more easier and quicker than learning some tool that already existed.
The benefits of writing your own tools are that you don't have to learn them, they will do exactly what you want them to do and you could potentially design them as simple and straightforward as possible from the very start, not mentioning about learning some new technologies while doing something new. The requirements for this particular system were for it to be easily portable over Unix/Linux/BSD/MacOS systems, hassle-free to kick start and straightforward simple to use. I've decided to develop [Ruby](https://www.ruby-lang.org/) based [DSL](https://en.wikipedia.org/wiki/Domain-specific_language) targeting **1.9.3** version of *Ruby* language as most of the operating systems either provide that version or a newer version and because I've already had experience developing *Ruby* based DSL to manage my ssh connections called [sshc](https://github.com/troydm/sshc/) which proved to be very portable and easy to use.
Note that I've also decided to not support Windows operating system as there isn't much benefit or need of doing that as I don't use it or know anyone who would want some dotfiles management system for it.
The result of this endeavor is in my git repo called [dotcentral](https://github.com/troydm/dotcentral/) however for learning experience I'm going to recreate it step by step for this blog post.
Also note that this repo contains my own dotfiles too alongside *Ruby* DSL configuration files used to automatically install them. So let's get started from scratch.

![Akutabe](https://i.imgur.com/CovRPWQ.png)

<!-- more -->

The system we are going to build is called **central** so let's create a file called **central** with following two lines and make it an executable using **chmod +x ./central**.
We don't have to specify an **.rb** extension for it as it's an [shebang][1] executable file.
Also we require two standard *Ruby* modules **erb** and **socket** which we'll use later in our functions.

{% codeblock lang:ruby %}
#!/usr/bin/env ruby
# encoding: utf-8

require 'erb'
require 'socket'
{% endcodeblock %}

This executable will be used to run specialized Ruby DSL files called configurations each describing some steps central script should take in order to install or configure some tool or dotfile.
Configuration files are usually named as **configuration.rb** but could be named any way you desire.
Common configurations are kept in common directory and private configurations are kept in private one but I also have each environment-specific configurations in their respective hostname based directories with their own **configuration.rb** files. Some applications with more complex dotfile structures like **vim** need more extensive additional steps so I put their configuration in their own directories to keep entire configuration structure simple and intuitive to navigate and manage.
Central script executes top-level **configuration.rb** file which executes all the necessary sub-directory **configuration.rb** files but it can also be used with only specific **configuration.rb** files as an arguments to *central* script in which case only that particular **configuration.rb** files will be executed.
Here is an top-level tree view of my own dotcentral repository containing my own **dotfiles** and **configuration.rb** files.
Also note some **dotfiles** have an **.erb** extension which I'll explain a little bit later.

{% codeblock %}

.
├── central
├── configuration.rb
└── configurations
    ├── common
    │   ├── bashrc
    │   ├── bashrc.erb
    │   ├── bin
    │   ├── configuration.rb
    │   ├── dircolors
    │   ├── fishrc
    │   ├── fishrc.erb
    │   ├── fonts
    │   ├── gitconfig
    │   ├── gitignore
    │   ├── mc
    │   ├── sshc
    │   ├── tigrc
    │   ├── tmux.conf
    │   ├── urxvt
    │   ├── vifm
    │   ├── vim
    │   ├── vimperator
    │   └── Xresources
    ├── private
    │   ├── common
    │   ├── configuration.rb
    │   ├── server
    │   ├── troymac.local
    │   ├── troynas
    │   └── troystick
    ├── server
    │   ├── bashrc.erb
    │   ├── bin
    │   ├── configuration.rb
    │   └── tmux.conf.erb
    ├── troymac.local
    │   ├── bashrc
    │   └── configuration.rb
    ├── troynas
    │   ├── bashrc
    │   ├── bashrc.erb
    │   ├── bin
    │   ├── configuration.rb
    │   ├── tmux.conf
    │   └── tmux.conf.erb
    └── troystick
        ├── compton.conf
        ├── configuration.rb
        ├── i3
        ├── i3status
        └── redshift.conf


{% endcodeblock %}

As we need to be aware of our environment or more precisely a hostname where the script is running let's create a function called **hostname**.
This function will be used in our configuration files.
Also we might want to execute different scenarios based on which operating system we are running so let's add an **os** function to easily determine the operating system name.

{% codeblock lang:ruby %}
# get hostname
def hostname
  Socket.gethostname
end

# get operating system
def os
  if RUBY_PLATFORM.include?('linux')
    return 'linux'
  elsif RUBY_PLATFORM.include?('darwin')
    return 'osx'
  elsif RUBY_PLATFORM.include?('freebsd')
    return 'freebsd'
  elsif RUBY_PLATFORM.include?('solaris')
    return 'solaris'
  end
end
{% endcodeblock %}

Next we need a way to run our **configuration.rb** files so we need to be aware of our working directory and be able to get absolute file path and file folder name.
So let's define functions for this.

{% codeblock lang:ruby %}
def pwd
  Dir.pwd
end

def abs(path)
  path = File.absolute_path(File.expand_path(path))
end

def chdir(dir)
  Dir.chdir(abs(dir))
end

def file_exists?(path)
  path = abs(path)
  File.file?(path) && File.readable?(path)
end

def dir_exists?(path)
  path = abs(path)
  Dir.exists?(path)
end

def file_dir(path)
  File.dirname(abs(path))
end
{% endcodeblock %}

In order to **run** our configuration file we need to temporally **chdir** into it's folder, load the *Ruby* file and then **chdir** back into previous working directory.
Also in some cases we might want to optionally run configuration files only if they exists so let's also define **run_if_exists** function.

{% codeblock lang:ruby %}
def run(file)
  cwd = pwd()
  file = abs(file)
  puts "Running configuration: "+file
  file_cwd = file_dir(file)
  chdir file_cwd
  load file
  chdir cwd
end

def run_if_exists(file)
  if file_exists?(file)
    run file
  end
end
{% endcodeblock %}

Now to run our configurations we are either going to execute the top-level one or the ones provided as arguments to *central* script.

{% codeblock lang:ruby %}
if ARGV.length > 0
  ARGV.map {|configuration| run configuration }
else
  run 'configuration.rb'
end
{% endcodeblock %}

Also since we need not only our common configuration but also our private one which is kept in a separate **git** repository we might as well add a capability to git clone/pull any repository we want from a configuration file.
But before doing that let's define a special function called **sudo** which will run any shell command with sudo prefix if command fails to run due to **permission denied** error.
This works only if you have sudo configured without a password and your configuration needs to access some files/executables which need *root* user privileges but isn't absolutely necessary if you access only your own files.
{% codeblock lang:ruby %}
def sudo(command,sudo=false)
  if sudo
    sudo = 'sudo '
  else
    sudo = ''
  end
  command = sudo+command
  out = `#{command} 2>&1`
  # removing line feed
  if out.length > 0 && out[-1].ord == 10
    out = out[0...-1]
  end
  # removing carriage return
  if out.length > 0 && out[-1].ord == 13
    out = out[0...-1]
  end
  if out.downcase.end_with?('permission denied')
    if sudo
      STDERR.puts "Couldn't execute #{command} due to permission denied\nrun central with sudo privileges"
      exit 1
    else
      out = sudo(command,true)
    end
  end
  return out
end
{% endcodeblock %}

Now let's add **git** function to clone any repository we might require and pull any repository already cloned to keep everything up-to date.
{% codeblock lang:ruby %}
def git(url,path)
  path = abs(path)
  if dir_exists?(path) && dir_exists?("#{path}/.git")
    cwd = pwd()
    chdir path
    out = sudo('git pull')
    unless out.downcase.include? "already up-to-date"
      puts out
      puts "Git repository pulled: #{url} → #{path}"
    end
    chdir cwd
  else
    out = sudo("git clone #{url} \"#{path}\"")
    puts out
    puts "Git repository cloned: #{url} → #{path}"
  end
end
{% endcodeblock %}

Before using **git** we need to check if it's actually installed in the system.
We might as well add checks for some other tools like **file**, **grep**, **curl** and **readlink** as we'll need them later in order to manage our configurations.

{% codeblock lang:ruby %}
# check all required tools
def checkTool(name,check)
  output = sudo("#{check} 2>&1 1>/dev/null").downcase
  if output.include?('command not found')
    STDERR.puts "#{name} not found, please install it to use central"
    exit 1
  end
end

checkTool('file','file --version')
checkTool('grep','grep --version')
checkTool('ln','ln --version')
checkTool('readlink','readlink --version')
checkTool('git','git --version')
checkTool('curl','curl --version')
{% endcodeblock %}

Now right about time for you to wonder how is this all used?
Following example is my top-level **configuration.rb** file.
First I git clone/pull my own private configuration repository.
Next I run it's **configuration.rb** file followed by running **common** **configuration.rb** file.
And finally I run **hostname** based configuration only if it exists for current host.

{% codeblock lang:ruby %}
git 'git@ubuntusrv:troydm/dotcentralprivate.git', 'configurations/private'
run 'configurations/private/configuration.rb'
run 'configurations/common/configuration.rb'
run_if_exists "configurations/#{hostname}/configuration.rb"
{% endcodeblock %}

Now let's try adding a symlink capability to our DSL in order to install [sshc](https://github.com/troydm/sshc) which I use everywhere.
So we need a function to manage symlinks and some minor functions to check if symlink exists and which path it points to.
So if file/dir exists in place where we want to create a symlink the system will stop with **exit 1** and **stderr** output will contain error description.
Otherwise we check the symlink path and if it's not valid one we delete it and create a symlink with specified new path.

{% codeblock lang:ruby %}
def symlink?(symlink)
  File.symlink?(abs(symlink))
end

def symlink_path(symlink)
  sudo("readlink \"#{abs(symlink)}\"")
end

def symlink(from,to)
  from = abs(from)
  to = abs(to)
  if symlink?(from)
    if symlink_path(from) != to
      rm from
      symlink from, to
    end
  elsif file_exists?(from)
    STDERR.puts "File #{from} exists in place of symlink..."
    exit 1
  elsif dir_exists?(from)
    STDERR.puts "Directory #{from} exists in place of symlink..."
    exit 1
  else
    out = sudo("ln -s \"#{to}\" \"#{from}\"")
    puts "Created symlink: #{from} → #{to}"
  end
end
{% endcodeblock %}

This is how it's used in my common **configuration.rb** to install sshc and make a symlink to it's executable in **bin** folder which I'll add to environment **PATH** via bashrc later.

{% codeblock lang:ruby %}
# install sshc
git 'https://github.com/troydm/sshc.git', 'sshc'
symlink 'bin/sshc', 'sshc/sshc'
{% endcodeblock %}

Let's add some more functions to our Ruby DSL to **mkdir** directories, **rm** files/directories, **touch** files and **chmod** files/directories.
You can easily add **chown** function too if you need it.
We'll need all these functions later in our configurations.

{% codeblock lang:ruby %}
def mkdir(path)
  path = abs(path)
  unless dir_exists?(path)
    out = sudo("mkdir -p \"#{path}\"")
    puts "Created directory: #{path}"
  end
end

def rm(path,recursive=false)
  path = abs(path)
  if recursive
    recursive = '-R '
  else
    recursive = ''
  end
  out = sudo("rm -#{recursive}f \"#{path}\"")
  puts "Removed file: #{path}"
end

def rmdir(path)
  rm(path,true)
end

def touch(path)
  path = abs(path)
  unless file_exists?(path)
    out = sudo("touch \"#{path}\"")
    puts "Touched file: #{path}"
  end
end

def chmod(path,permissions,recursive=false)
  path = abs(path)
  if recursive
    recursive = '-R '
  else
    recursive = ''
  end
  sudo("chmod #{recursive}#{permissions} \"#{path}\"")
end
{% endcodeblock %}

We'll also need **curl** function in order to download files.

{% codeblock lang:ruby %}
def curl(url,path)
  path = abs(path)
  output = sudo("curl -s \"#{url}\"")
  unless $?.exitstatus == 0
    STDERR.puts "Couldn't download file from #{url}..."
    exit 1
  end
  if File.exists?(path) && File.read(path) == output
    return
  end
  File.write(path,output)
  puts "Downloaded #{url} → #{path}"
end
{% endcodeblock %}

Now that we have **curl** and **chmod** we can easily install [ack](https://beyondgrep.com/) via our configuration file.

{% codeblock lang:ruby %}
# install ack
curl 'http://beyondgrep.com/ack-2.14-single-file', 'bin/ack'
chmod 'bin/ack', '0755'
{% endcodeblock %}

Next we'll need **ls** function in our *Ruby* DSL which will support multiple options via second **Hash** type argument such as filtering of file/directory names based on **grep** pattern and listing either only files or directories as well as including dotfiles if necessary.
Quite complicated function indeed, however this is necessary because of how we usually need to use it in our configurations depending on whenever we need to list only files or directories or some specific files or directories based on some predefined pattern.

{% codeblock lang:ruby %}
def ls(path,options={})
  path = abs(path)
  if options[:dotfiles]
    dotfiles = '-a '
  else
    dotfiles = ''
  end
  command = "ls -1 #{dotfiles}\"#{path}\""
  if options.key?(:grep) && options[:grep].length > 0
    command += " | grep #{options[:grep]}"
  end
  output = sudo(command)
  if output.downcase.end_with?('no such file or directory')
    STDERR.puts "Couldn't ls directory #{path}..."
    exit 1
  end
  ls = output.split("\n")
  dir = true
  file = true
  if options.key?(:dir)
    dir = options[:dir]
  end
  if options.key?(:file)
    file = options[:file]
  end
  unless dir
    ls = ls.keep_if {|f| !File.directory?("#{path}/#{f}") }
  end
  unless file
    ls = ls.keep_if {|f| !File.file?("#{path}/#{f}") }
  end
  return ls
end
{% endcodeblock %}

With the help of **ls** now we can finally install Powerline fonts.
The code below clones git repository and installs all **.ttf** and **.otf** symlinks into either ~/.fonts or ~/Library/Fonts directory based on which operating system you are on.

{% codeblock lang:ruby %}
# install Powerline fonts on OSX and Linux only
if os == 'linux'
    install_fontsdir = '~/.fonts'
elsif os == 'osx'
    install_fontsdir = '~/Library/Fonts'
end
if install_fontsdir
  git 'https://github.com/powerline/fonts.git', 'fonts'
  mkdir install_fontsdir
  ls('fonts',{:file => false}).each { |fontdir|
      ls("fonts/#{fontdir}",{:grep => '.[ot]tf'}).each { |font|
          symlink "#{install_fontsdir}/#{font}", "fonts/#{fontdir}/#{font}"
      }
  }
end
{% endcodeblock %}

At this point we need a way to manage some differences that might arise from different operating systems and deployment path inside our dotfiles itself.
For this we need some kind of templating system which will allow us to include or dynamically generate files when processing configuration files.
Fortunately *Ruby* has [erb](http://www.stuartellis.name/articles/erb/) templating system so we don't have to write our own. 
This will allow us to generate dotfiles on the fly as well as include content of some files into another ones.

{% codeblock lang:ruby %}
def erb(file,output_file = nil)
  file = abs(file)
  if output_file == nil
    if file.end_with?('.erb')
      output_file = file[0...-4]
    else
      output_file = file+'.out'
    end
  end
  if file_exists?(file)
    output = ERB.new(File.read(file)).result
    if File.exists?(output_file) && File.read(output_file) == output
      return
    end
    File.write(output_file,output)
    puts "Processed erb #{file} → #{output_file}"
  else
    STDERR.puts "Couldn't process erb file #{file}..."
    exit 1
  end
end
{% endcodeblock %}

In order to include files we'll need a small function to be able to easily read them.

{% codeblock lang:ruby %}
def read(file)
  file = abs(file)
  if file_exists?(file)
    return File.read(file)
  else
    STDERR.puts "Couldn't read file #{file}..."
    exit 1
  end
end
{% endcodeblock %}

This is for example how I use this function to centrally manage my tmux configuration in **tmux.conf.erb** file.
It includes common **tmux.conf** into my host-specific **tmux.conf.erb**.

{% codeblock lang:ruby %}
<%= read '../common/tmux.conf' %>

# Left status bar
set -g status-left " "

# Right status bar
set -g status-interval 1
set -g status-right-length 200
{% endcodeblock %}

This **tmux.conf.erb** file is later processed from **configuration.rb** file resulting into **tmux.conf** file being generated.
{% codeblock lang:ruby %}
erb 'tmux.conf.erb'
symlink '~/.tmux.conf', 'tmux.conf'
{% endcodeblock %}

And finally to manage shell configuration I want to source my bashrc into ~/.bashrc file in an automated way.
To do this we'll create a function called **source**. It'll add source line to any shell configuration file if it isn't already added there.

{% codeblock lang:ruby %}
def source(file,source)
  file = abs(file)
  source = abs(source)
  source_line = "source \"#{source}\""
  out = sudo("grep -Fx '#{source_line}' \"#{file}\"")
  if out == ""
    sudo("echo '#{source_line}' >> \"#{file}\"")
    puts "Added source #{source} line to #{file}"
  end
end
{% endcodeblock %}

Now finally I can use this function to source my configuration specific **bashrc** file which is generated from **bashrc.erb** into **~/.bashrc**
{% codeblock lang:ruby %}
erb 'bashrc.erb'
source '~/.bashrc', 'bashrc'
{% endcodeblock %}

That's it folks this how I manage my dotfiles mess. I hope you learned something new today. Have a good coding time!

![Azazel-san](https://i.imgur.com/45SwyJm.png)

[1]: https://en.wikipedia.org/wiki/Shebang_(Unix)
