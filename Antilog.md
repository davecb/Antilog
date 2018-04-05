# Sherlock Homes on Log Files

At customer after customer, whenever something goes wrong, we get asked to look at the logs and tell them what they mean.

This is an excellent way to roll up a large number of consulting hours, but it isn't what one really wants to do, unless you <span style="font-style: normal">enjoy</span> grovelling over logs for hours, hoping to recognize something that will point you to the problem. Automated tools suffer from the same problem as manual log browsing: you don't know what you're looking for until after you've found it.

However, there is a better way, as Sherlock Holmes pointed out in _The Adventure of the Beryl Coronet_.

> It is an old maxim of mine that when you have excluded the impossible, whatever remains, however improbable, must be the truth

If we exclude the uninteresting, regular or routine, then whatever remains will be extraordinary, and likely to to be relevant to the problem we are trying to diagnose.

Fortunately, we can exclude routine log entries quite easily. We take all the old logs, up to but not including the day the problem ocurred, remove the date-stamps, and use sort -u to create a table of log entries that we've seeen peviously. We then turn tehe table into an awk program to filter out these lines, and run that against the logs from the day the problem ocurred.

For example, a machine named “froggy” had a problem with Sun's update program, and the /var/adm/messages contents for previous days, once sorted, were a series of lines like this:

<pre class="western">froggy /sbin/dhcpagent[1007]: [ID 929444 daemon.warning] configure_if: no IP broadcast specified for dmfe0, making best guess
froggy /sbin/dhcpagent[1029]: [ID 929444 daemon.warning] configure_if: no IP broadcast specified for dmfe0, making best guess
froggy gconfd (root-776): [ID 702911 user.error] file gerror.c: line 225: assertion `src != NULL' failed
froggy pcipsy: [ID 781074 kern.warning] WARNING: pcipsy0: spurious interrupt from ino 0xb
</pre>

A bit of editing turned this into

<pre class="western">/dhcpagent.* configure_if: no IP broadcast specified/ { next; }
<span lang="en-GB">/gconfd.* file gerror.c: line 225/ { next; }</span>
<span lang="en-GB">/pcipsy.* WARNING: pcipsy0: spurious interrupt from/ next; }</span>
<span lang="en-GB">/.*/ { print $0}</span></pre>

That's an awk program which will remove all variants on these three error messages. When the program was extended to a mere 23 lines and run against the current day's /var/adm/messages, it exposed the following error:

<pre class="western">froggy CNS Transport[1160]: [ID 195303 daemon.error] unhandled exception: buildSysidentInfo: IP address getSystemIPAddr error: node name or service name not known</pre>

A little checking showed us that this was the Sun Update Connection program complaining that there was not network available, which was perfectly reasonable, as the machine in question was a laptop and really wasn't connected to a network at the time the error occurred.

So we added a rule for updating-program errors, right after the one for “no IP broadcast specified”, another message that only occurs when the machine was disconnected.

### Turning This into a Process and a Tool

This kind of manual programming is fine for a on-off exercise, but it would be better to have a config file we can update occasionally, and run the filter program every night against every machine via cron. Then any new messages would be reported bright and early for us to look at, before the users discover it.

To make it easier, and to reduce the likelihood of making a typographical error creating the patterns, we wrote an awk program to translate a configuration file into proper awk patterns, and called it antilog. To give you an easy starting point, we put a default config file into the program itself, so you can just copy it out and add local additions.

In addition, we taught it a bit more about syslog log files, as they're very common, and added a few options for debugging and for emailing the results to an individual or list.

Finally, we invented a notation for multiple lines which begin with the same pattern:

<pre class="western">daemon.error {
        Failed to monitor the power button.
        Unable to connect to open /dev/pm: pm-actions disabled
}</pre>

This will create two awk commands of the form

<pre class="western">	daemon.error.*Failed to monitor the power button.
	daemon.error.*Unable to connect to open /dev/pm: pm-actions disabled</pre>

to save typing (and typos).

Our final config file looks like this:

<pre class="western"># Boot time
last message repeated
Use is subject to license terms.
auth.crit {
        rebooted by root
}

# Power management
daemon.error {
        Failed to monitor the power button.
        Unable to connect to open /dev/pm: pm-actions disabled
}

# sendfail(8) blunders
mail.alert] unable to qualify my own domain name
mail.crit] My unqualified host name

# Shutdown
syslogd: going down on signal 15
daemon.error] rpcbind terminating on signal.

# DHCP
daemon.warning] configure_if: no IP broadcast specified

# Networking
NOTICE.*TX stall detected after: pass
kern.warning] WARNING: pcipsy0: spurious interrupt from ino

# Gconfd
user.error] file gerror.c. line 225\. assertion

# Java
user.error {
        libpkcs11: /usr/lib/security/pkcs11_softtoken.so unexpected failure
        libpkcs11: open /var/run/kcfd_door: No such file or directory
}

# And trim off the rest of the low-priority kernel messages.
kern.notice
kern.info
</pre>

When run, Antilog compiles the program above into:

<pre class="western">/\.debug\]/ { next }
/kern.notice/ { next }
/kern.info/ { next }
/last message repeated/ { next }
/Use is subject to license terms./ { next }
/auth.crit.*rebooted by root/ { next }
/daemon.error.*Failed to monitor the power button./ { next }
/daemon.error.*Unable to connect to open \/dev\/pm: pm-actions disabled/ { next }
/mail.alert\] unable to qualify my own domain name/ { next }
/mail.crit\] My unqualified host name/ { next }
/syslogd: going down on signal 15/ { next }
/daemon.error\] rpcbind terminating on signal./ { next }
/daemon.warning\] configure_if: no IP broadcast specified/ { next }
/NOTICE.*TX stall detected after:/ { print $0; next }
/kern.warning\] WARNING: pcipsy0: spurious interrupt from ino 0x2a/ { next }
/user.error\] file gerror.c. line 225\. assertion/ { next }
/user.error.*libpkcs11: \/usr\/lib\/security\/pkcs11_softtoken.so/ { next }
/user.error.*libpkcs11: open \/var\/run\/kcfd_door: No such file/ { next }
/kern.notice/ { next }
/kern.info/ { next }
/.*/ { print $0 }
</pre>

We've used this same technique quite a number of times, for syslog files after an Oracle crash, warning messages from a compiler and log file from the X.org release of the X Window System, In each case it's turned hours of reading into a few minutes of preprocessing.
