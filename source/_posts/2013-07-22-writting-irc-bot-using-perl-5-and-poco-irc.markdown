---
layout: post
title: "Writting IRC Bot using Perl 5 and POCO::IRC"
date: 2013-07-22 21:29
comments: true
categories: [perl, irc, shell]
---

Some people use [IRC](https://en.wikipedia.org/wiki/Internet_Relay_Chat) to chat, some don't. 
It was invented a really long time ago and isn't going away anytime soon despite some new generation alternatives poping up like [Jabber](http://www.jabber.org/). 

Personally i always have my irc client running (i'm using [weechat](http://weechat.org/) + [tmux](http://tmux.sourceforge.net/)) and chat with lots of interesting people
who inspire me to try new technologies and learn something different every day. One person, who's nick i won't name, was always telling me about how awesome [Perl][1] 
as a programming language is and how great it's potential is thanks to [CPAN][4] that has almost 124k modules for any life situation. 
I always thought he was exaggerating and literally acting like a [Perl][1] fanboy. [Perl][1] was the first programming language i've learned back in the late 90's 
and remembering how frustuating my experience with it was and how cryptic it really was for me do something with it when i was unexperienced and lacked lots of qualities that make up a 
any decent software engineer i was skeptic about using it again. 
Well, time passed, time always passes, and i haven't written anything more than quick 50 line server scripts in [Perl][1] for almost 13 years.
I've almost forgotten everything about [Perl][1]. Since lately i was having this crazy idea about writing irc bot that could store and execute shell scripts on server
so i could automate my servers through irc, i thought why not write it in [Perl][1]. I've remembered that person who was always bragging about [Perl][1]'s greatness wrote an irc bot in [Perl][1]
using [POE::Component::IRC][2] so i've decided to try and use the same framework for my bot. It's based on really popular [POE][3] event loop framework which is very easy to learn and use.
**Matt Cashner** wrote a really good introduction article called [Application Design with POE](http://www.perl.com/pub/2004/07/02/poeintro.html)

The whole code for my bot is just 500 lines and is available from this repository [shellbot](https://github.com/troydm/shellbot).
I'm going to walk through a key concepts that are essential for writing an irc bot in [POCO::IRC][2] using my bot's source code as a reference.

![Chobits](http://i.imgur.com/kiqhDBH.jpg)

Before we'll start our [Perl][1] IRC bot we need some way to store configuration for it. Since [CPAN][4] has lots of modules that deal with configuration the choice wasn't an easy one but 
i've decided to use [YAML](https://metacpan.org/module/YAML) which is module for loading [YAML][5] data into [Perl][1] that can work the other way too. [YAML][5] is a simple markup language
that is perfect for storing configuration and it's really quick to learn. For loading YAML configuration i've used **LoadFile** function and to store [Perl][1] data back into file i've used **DumpFile** function.
Just two simple functions that do all the complex work work for me. Since i wanted to store commands in the same file i just used [Perl][1]'s list construct to specify that i'm loading two seperate [YAML][5] documents.

{% codeblock lang:perl %}
# Loading configuration 
my ($config, $commands) = LoadFile($config_file) 
    || die "invalid configuration file specified";

# Storing configuration
DumpFile($config_file, ($config, $commands));
{% endcodeblock %}

Next step is to create POE Session and start event loop. Note that since my module is named Shellbot i need to specify it otherwise event loop won't be able to call functions.
Each of this functions are called by [POE][3] when specified events occur and all the bot logic is handled by those functions.

{% codeblock lang:perl %}
POE::Session->create(
    package_states => [
        Shellbot => [ qw(_start _default irc_join irc_msg irc_bot_addressed irc_connected 
                         got_job_stdout got_job_stderr got_job_close got_child_signal) ]
    ]
);

POE::Kernel->run();
{% endcodeblock %}

First event that is executed after the POE event loop starts is **_start** event so we need to initialize bot
state in that event. Also since [POCO::IRC][2] comes with some essential plugins and is modular by itself we can take advantage of this.
Instead of manually making bot reconnect when it looses connection with server i've used [Connector](https://metacpan.org/module/POE::Component::IRC::Plugin::Connector) plugin.
If we want our bot to automaticly join some channel we can use [AutoJoin](https://metacpan.org/module/POE::Component::IRC::Plugin::AutoJoin) plugin.
And to easily handle when someone addresses bot i've used [BotAddressed](https://metacpan.org/module/POE::Component::IRC::Plugin::BotAddressed) plugin. 
Since i wanted my bot to handle commands i could have used [BotCommand](https://metacpan.org/module/POE::Component::IRC::Plugin::BotCommand) plugin however i wanted my bot 
to have two modes of commands so i've decided to write bot command handling functions manually.

{% codeblock lang:perl %}
my $irc = POE::Component::IRC::State->spawn(%opts);
$heap->{irc} = $irc;

# Connector plugin
$heap->{connector} = POE::Component::IRC::Plugin::Connector->new();
$irc->plugin_add( 'Connector' => $heap->{connector} );

# Autojoin plugin
if(exists($config->{'channels'}) && $config->{'channels'} > 0){
    $irc->plugin_add('AutoJoin', POE::Component::IRC::Plugin::AutoJoin->new(
       Channels => $config->{'channels'}
    ));
}

# BotAddressed plugin
$irc->plugin_add( 'BotAddressed', POE::Component::IRC::Plugin::BotAddressed->new() );

$irc->yield(register => qw(join msg connected));
$irc->yield('connect');
{% endcodeblock %}

Since bot can be addressed both using a private message and refering him on a channel i've decided to handle 
both **irc_msg** and **irc_bot_addressed** events uniformly in **msg_received** function

{% codeblock lang:perl %}
sub irc_msg {
    my $irc = $_[SENDER]->get_heap();
    my @nick = ( split /!/, $_[ARG0] );
    my $msg = $_[ARG2];
    my $heap = $_[HEAP];
    my $kernel = $_[KERNEL];
 
    msg_received $irc, $heap, $kernel, $nick[0], $nick[1], '', $msg;
}

sub irc_bot_addressed {
    my $irc = $_[SENDER]->get_heap();
    my @nick = ( split /!/, $_[ARG0] );
    my $channel = $_[ARG1]->[0];
    my $msg = $_[ARG2];
    my $heap = $_[HEAP];
    my $kernel = $_[KERNEL];
 
    msg_received $irc, $heap, $kernel, $nick[0], $nick[1], $channel, $msg;
}
{% endcodeblock %}

Before bot accepts a command we need some way to check if person who is issuing a command is authorized to do so. I'm doing a simple check of full irc name
that can be specified in configuration list option **authorizations**

{% codeblock lang:perl %}
sub is_authorized {
    my $name = $_[0];
    return grep { $_ eq $name } @{ $config->{'authorizations'} };
}
{% endcodeblock %}

All that is left is to match commands in **msg_recieved** using [Perl][1]'s [regex](http://perldoc.perl.org/perlre.html). This part is just a long series of bot command logic
and **if** **elsif** **else** statements so i won't reference them here. Also the key function is **run_job** which executes prestored shell script. It creates a temporary file that
is executed using a shell

{% codeblock lang:perl %}
my $program = "/bin/bash";
if(exists($config->{'shell'})){
    $program = $config->{'shell'};
}

# create a temporary script
my $script = File::Temp->new();
my $scriptname = $script->filename;
chmod 0700, $script;
if(ref $cmd eq 'ARRAY'){
    foreach my $line (@{ $cmd }){
        print $script "$line\n";
    }
}else{
    print $script $cmd;
}
$program .= " $scriptname $args";
{% endcodeblock %}

Shell execution is handled by [POE][2] using [POE::Wheel::Run](https://metacpan.org/module/POE::Wheel::Run) 
and output from shell is received on **got_job_stdout** and **got_job_stderr** functions
{% codeblock lang:perl %}
my $job = POE::Wheel::Run->new(
    Program      => $program,
    StdioFilter  => POE::Filter::Line->new(),
    StderrFilter => POE::Filter::Line->new(),
    StdoutEvent  => "got_job_stdout",
    StderrEvent  => "got_job_stderr",
    CloseEvent   => "got_job_close",
);
{% endcodeblock %}

That's all there is to it. To install and trying it out just look through a [readme](https://github.com/troydm/shellbot/blob/master/README.md)

Offcourse making a temporary shell script and executing it is generally unsafe. Also there is security concern
that bot can be somehow hacked and commanded by unathorized nick but i'm not sure how can this be done without changing vhost.
That is why i called this bot a potentially unsafe. If anyone can find any security holes please do pull request or just email me patch.

To make this bot little more secure we need to make him connect to irc using SSL. For this i'll walk through a general steps for configuring
[CertFP](https://www.freenode.net/certfp/) and generating self signed certificate for [Freenode](https://www.freenode.net/) as an example.
First step is to register your bot's nick with [NickServ](https://blog.freenode.net/2007/03/nickserv-is-your-friend/). 
Choose a nickname for your bot and start it up. Ask your bot to register with NickServ by private messaging him on irc.
{% codeblock %}
/msg botnick msg NickServ REGISTER nickservpass email@address.com
{% endcodeblock %}
Shortly you'll receive an email verification that will include verification code, privately message your bot again to verify your registration
{% codeblock %}
/msg botnick msg NickServ VERIFY REGISTER botnick verificationcode
{% endcodeblock %}
Now your bot's nick is registered with NickServ so edit your configuration file and uncomment nickserv line to specify your nickservpass that will
be used each time your bot will connect to server for authorization of your nick

Now we'll generate a self signed certificate for SSL, i've used [this](https://www.freenode.net/certfp/makecert.shtml) manual for reference.
{% codeblock %}
umask 077
openssl req -newkey rsa:2048 -days 730 -x509 -keyout botnick.key -out botnick.crt
{% endcodeblock %}
You'll be asked some questions and after that openssl will generate two files, your bot's certificate and key.
For CertFP to work we need a fingerprint of your key and certificate
{% codeblock %}
cat botnick.crt botnick.key > botnick.pem
openssl x509 -sha1 -noout -fingerprint -in botnick.pem | sed -e 's/^.*=//;s/://g;y/ABCDEF/abcdef/' 
{% endcodeblock %}
After this you'll get a fingerprint that you need to add to NickServ
{% codeblock %}
/msg botnick msg NickServ CERT ADD fingerprint
{% endcodeblock %}

Now all that is left is to specify in configuration that we are connecting using SSL certificate and key.
Just uncomment **sslcrt** and **sslkey** options and specify full path to your bot's certificate and key.
Also don't forget to change the port since we need to specify an SSL port instead of usual one, just specify 
**6697**, **7000** or **7070** and restart your bot to enjoy fully secured connection to irc server.
Also i've used [Proc::Daemon](https://metacpan.org/module/DETI/Proc-Daemon-0.14/lib/Proc/Daemon.pod) for running this bot
as daemon so you can uncomment **daemon** and **log** options to run it as daemon process.


So what did i learned from writing this small irc bot in [Perl][1]?
Now the first most common mistake i was always making when coding in [Perl][1] was the difference between list and array, so
I recommend this article by **Mike Friedman** [Arrays vs. Lists in Perl](http://friedo.com/blog/2013/07/arrays-vs-lists-in-perl)
that helped me a lot. Also dereferncing was the second most common mistake i had, until i've read this article 
[Dereferencing in Perl](http://perlmeme.org/howtos/using_perl/dereferencing.html). But the biggest thing i've learned was that [Perl][1]
as a scripting language is really easy to use and powerfull despite some people claiming it's dead, it's still alive and it's doing 
just [fine](http://www.nntp.perl.org/group/perl.perl5.porters/2013/07/msg204905.html)! 
Just look how many modules are released on [CPAN][4] everyday! And it's really really fun to code in, so everyone should try learning and using it!!!



[1]: http://perl.org/ "Perl"
[2]: https://metacpan.org/module/POE::Component::IRC "POCO::IRC"
[3]: https://metacpan.org/module/POE "POE"
[4]: http://cpan.org "CPAN"
[5]: http://yaml.org "YAML"
