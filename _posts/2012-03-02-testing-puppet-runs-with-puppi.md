---
layout: blog
title: Testing Puppet runs with Puppi
created: 1330680858
---
<p>Whoever has used Puppet on production servers has probably lived, at least once, the uncomfortable situation when a change on Puppet manifests has introduced "unwanted effects" on target nodes if not real failures.<br />There are many things that can go wrong when you make a change in whatever part of your Puppet code: you may introduce a wrong configuration file that triggers a service that doesn't restart correctly or have a modification you expected to be limited to some servers and, for some reason, due to wrong logic or poor understanding of your Puppet code, be applied to the "wrong" servers.<br />Then there are other more mundane errors, that might occur with syntax errors that prevent Puppet from running on a node, but do not affect its functionality. </p><p>Let's see a brief summary of the common errors and the testing methods available for Puppet.</p><h2>Common errors and testing methods</h2><p><strong>Syntax errors </strong>are quite common for the unexperienced Puppet Master still battling with Puppet's somehow picky syntax, but they are also the <strong>less dangerous </strong>and <strong>easier to test</strong>. It's a very good idea to force syntax checks, both on Puppet manifests and Erb templates directly at the commit stage, with an <strong>hook command </strong>of your favorite SCM (search for something like "<em>puppet syntax check hook</em>" for implementation details). Another somehow unusual but effective method to test the manifests syntax is to use <strong>Puppet doc</strong>. A simple command like:</p><pre> puppet doc --outputdir /var/www/doc/ --mode rdoc</pre><p>not only generates a nice Rdoc output like the one <a title="PuppetDoc NextGet" href="http://www.example42.com/puppetdoc/modules-nextgen/">here</a> but does a syntax check for all your manifests.<br />Moreover, if you like to code with an IDE, <a title="Geppetto" href="http://cloudsmith.github.com/geppetto/">Geppetto</a> is definitively your choice, having, among the others, also syntax checking and autocompletion features.</p><p><strong>Modules' integration and logic errors </strong>may be intercepted by proper tests. <strong><a href="http://projects.puppetlabs.com/projects/cucumber-puppet" title="Cucumber Puppet">Cucumber Puppet</a> </strong>and <strong><a href="https://github.com/rodjek/rspec-puppet" title="Rspec Puppet">Rspec Puppet</a> </strong>introduce Puppet to some of the fancier testing methodologies around. They examine the catalog that would be produced for an host and test if it contains the expected resources. As you can imagine their effectiveness is strictly related to the quality of the tests you write and a good coverage of your classes and defines requires time.</p><p>I must confess that I was initially skeptical about their usefulness, but when I started to use rspec puppet I've found it really useful to catch regressions, integration and logic issues and found myself much more comfortable in committing changes to my modules set.</p><p>The puppet <strong>--noop</strong> option is also quite useful to test if the catalog provides the expected resources for a node, but requires manual intervention.</p><p><strong>Service failures</strong>, though, can be due to other kind of errors, that can't be intercepted just by checking if Puppet compiles its catalog correctly: an error in a configuration file that prevents the relevant service from restarting can be noticed only when it has already been applied to a node. Also applying a syntactically correct configuration on the wrong nodes may have terrible effects (imagine how <em>bad&#160;</em>could be to apply the apache configuration reserved to a specific kind of servers to nodes that should have a different role...)<br />To mitigate or prevent the impact of this kind of problems there are different approaches, none of them is really resolutive but still have their usefulness:<br />- You may use <strong>Puppet's environments </strong>to test the effect of a change on a limited set of nodes before propagating it to the whole infrastructure. An implementation example is <a title="Git Workflow and Puppet Environments" href="http://puppetlabs.com/blog/git-workflow-and-puppet-environments/">here</a>. <br />- You can use <strong>canary nodes</strong>, that can eventually be created from scratch and destroyed each time, where your configurations are applied and their effect verified. A nice sample with Vagrant is shown <a title="Automated virtual test-environments with Vagrant and Puppet" href="http://blog.codecentric.de/en/2012/02/automated-virtual-test-environments-with-vagrant-and-puppet/">here</a>.<br />- You have probably already <strong>testing and staging environments</strong>, in your infrastructure, where you test your applications before deploying them in production and where you can test also your Puppet changes, even if you might not be 100% sure that the Puppet settings for your testing nodes are the same of your production nodes.<br />- You can introduce in your change workflow a <strong>code peer review </strong>phase, with tools like <a title="Gerrit" href="http://code.google.com/p/gerrit/">Gerrit</a>, that ease team communication and limit the involuntary disasters that can be introduced by Puppet newbies.<br />- You can, also, use <strong>Puppi </strong>as <strong>postrun command </strong>and be notified if your Puppet run has done something wrong.</p><p>And this is actually what I'm going to describe in this post.</p><h2>Running puppi check to test Puppet runs</h2><p>Puppi has various actions, among them I find myself using quite frequently, <strong>puppi check</strong>.</p><p>It runs a series of checks on your system, its running services and eventually its web applications (or actually whatever can be checked by a nagios plugin) and shows in a quick way if everything is running fine. This is useful when you are logged to a node, to quickly see what's wrong in it, or when you deploy an application to verify immediately if you've made some damage, or via mcollective, to check quickly a cluster of nodes, or, as we are going to see, after a Puppet run, to be notified immediately if something has gone wrong.</p><p>To use puppi you just have to include it in your nodes or in a class that is included by your node:</p><pre>include puppi</pre><p>To be sure to have all the commands Puppi needs include also the puppi::prerequisites class (that works for Example42 modules) or to be sure to have all the resources provided there:</p><pre>include puppi::prerequisites</pre><p>Once puppi is on your system you can play a bit with it, by issuing commands like <strong>puppi info</strong>, <strong>puppi log</strong> or <strong>puppi check</strong> (nagios plugins should be installed on your system).</p><p>By default the quality and the richness of their output is rather limited, but,&#160;if you use the Example42 modules set you can add automatic Puppi integration just setting these variables:</p><pre>$puppi = yes   # Enables puppi integration.
$monitor = yes # Enables automatic monitoring 
$monitor_tool = "puppi" # Sets puppi as monitoring tool</pre><p>Note that at this point you might extend the monitoring tools to use, to enable automatic checks on the relevant tools. For example you can set:</p><pre>$monitor_tool = ["puppi","nagios","munin"]</pre><p>If you don't use Example42 modules or have some modules you want to check with puppi you can just add defines like these (either in the relevant modules on in other classes):</p><pre>    puppi::check { "apache":
      command  =&gt; "check_tcp -H $fqdn -p 80",
    }</pre><p>the above checks for port 80 on the local system using the check_tcp Nagios plugin, to check if a process is running you can use something like:</p><pre>    puppi::check { "apache_process":
      command  =&gt; "check_procs -c 1: -C httpd",
    }</pre><p>Needless to say that if some of these values are present in your module as variables, you can parametrize them.</p><p>You can use also custom scripts for these checks, they just have to be in the Nagios plugins directory of your server (this might be fixed soon, giving the possibility of specifying an absolute custom path) and have an exit code logic compliant with them: Exit 0 if everything is OK, exit 1 for WARNING, exit 2 for CRITICAL.</p><p>Whatever you do, the checks you define for your node are shown with the puppi check command.</p><p>Now you can configure Puppet to run these checks at the end of each <strong>Puppet run</strong> and be notified via mail if any of them FAILS (exit code 2). You just have to add something like this in the [main] stanza of your puppet.conf.</p><pre>postrun_command = "/usr/bin/mailpuppicheck -m alerts@mycompany.com"</pre><p>The mailpuppicheck is a rather naif script provided by the Puppi module, it has the following options:</p><pre>-m email_address # Specify the destination email for alerts
-r 4 # Number of times a puppi check is re-run in case of failures</pre><p>The -r option (default value is 1) can be useful if you want to re-run a puppi check more times, in case of failures, before actually sending an email. Note that if this value is not low and a puppi check run takes some time, especially if there are failures with long timeout, your puppet run can last more than you might like.</p><p>In order to avoid, too much spamming, the mailpuppicheck command doesn't &#160;send an alert email if the failures encountered are the same of the previous Puppet run. In this way you are not regularly notified of persistent problems.</p><p>More options to better manage notification logic might be added in future, the same notification method, email, is quite primitive ( I personally hate it ) and can be changed to something more interesting in the future.</p><p>The result, in any case is a mail like:</p><pre>Subject:&#160; [puppet] Errors after Puppet run on web01.example42.com
web01 check: 50-apache_process FAILED
web01 check: 50-apache_tcp_80 FAILED</pre><p>which is sent immediately after a Puppet run and can give early warning of incoming disasters, especially if you have a battery of servers where the puppet run is sprayed on some time interval.</p><p>So, this is far from being the definitive solution to testing the impact of Puppet changes on your infrastructure, also because it happens AFTER the change has been applied, when it is late but maybe not too late, but is a useful layer of "early warning".</p><p>Moreover, if you already use Example42 modules with the above settings, all this comes out free, with no extra effort: you add a module and that module's process and port is automatically checked. For example, on the server running this site, this is the output of puppi check, obtained out of the box.</p>
<pre>[root@web01 ~]# puppi check
web01 check: 10-Connected_Users                            [  OK  ]
USERS OK - 1 users currently logged in |users=1;5;10;0

web01 check: 10-Disks_Usage                                [  OK  ]
DISK OK - free space: / 27771 MB (86% inode=95%); /dev/shm 1915 MB (100% inode=99%); /boot 425 MB (92% inode=99%);| /=4405MB;27118;30508;0;33898 /dev/shm=0MB;1532;1723;0;1915 /boot=33MB;387;435;0;484

web01 check: 10-Local_Mail_Queue                           [  OK  ]
OK: mailq is empty|unsent=0;2;5;0

web01 check: 10-System_Load                                [  OK  ]
OK - load average: 0.05, 0.02, 0.00|load1=0.050;15.000;30.000;0; load5=0.020;10.000;25.000;0; load15=0.000;5.000;20.000;0; 

web01 check: 10-Zombie_Processes                           [  OK  ]
PROCS OK: 0 processes with STATE = Z

web01 check: 15-DNS_Resolution                             [  OK  ]
DNS OK: 0.004 seconds response time. example.com returns 192.0.43.10|time=0.004123s;;;0.000000

web01 check: 20-NTP_Sync                                   [  OK  ]
NTP OK: Offset 0.000167965889 secs|offset=0.000168s;60.000000;120.000000;

web01 check: 50-apache_process                             [  OK  ]
PROCS OK: 21 processes with command name 'httpd'

web01 check: 50-apache_tcp_80                              [  OK  ]
TCP OK - 0.000 second response time on port 80|time=0.000136s;;;0.000000;10.000000

web01 check: 50-cron_process                               [  OK  ]
PROCS OK: 1 process with command name 'crond'

web01 check: 50-munin_process                              [  OK  ]
PROCS OK: 1 process with command name 'munin-node'

web01 check: 50-munin_tcp_4949                             [  OK  ]
TCP OK - 0.001 second response time on port 4949|time=0.001250s;;;0.000000;10.000000

web01 check: 50-mysql_process                              [  OK  ]
PROCS OK: 1 process with command name 'mysqld'

web01 check: 50-mysql_tcp_3306                             [  OK  ]
TCP OK - 0.000 second response time on port 3306|time=0.000111s;;;0.000000;10.000000

web01 check: 50-nrpe_process                               [  OK  ]
PROCS OK: 1 process with command name 'nrpe'

web01 check: 50-nrpe_tcp_5666                              [  OK  ]
TCP OK - 0.000 second response time on port 5666|time=0.000276s;;;0.000000;10.000000

web01 check: 50-ntp_process                                [  OK  ]
PROCS OK: 1 process with command name 'ntpd'

web01 check: 50-openssh_process                            [  OK  ]
PROCS OK: 2 processes with command name 'sshd'

web01 check: 50-openssh_tcp_22                             [  OK  ]
TCP OK - 0.000 second response time on port 22|time=0.000373s;;;0.000000;10.000000

web01 check: 50-postfix_process                            [  OK  ]
PROCS OK: 1 process with command name 'master'

web01 check: 50-postfix_tcp_25                             [  OK  ]
TCP OK - 0.000 second response time on port 25|time=0.000178s;;;0.000000;10.000000

web01 check: 50-rsyslog_process                            [  OK  ]
PROCS OK: 1 process with command name 'rsyslogd'

web01 check: 50-splunk_process                             [  OK  ]
PROCS OK: 2 processes with command name 'splunkd'

web01 check: 50-splunk_tcp_8089                            [  OK  ]
TCP OK - 0.000 second response time on port 8089|time=0.000119s;;;0.000000;10.000000
</pre><p>my2c ;-)&#160;</p>
