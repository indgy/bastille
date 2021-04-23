========
Template
========
Looking for ready made CI/CD validated [Bastille
Templates](https://gitlab.com/BastilleBSD-Templates)?

Bastille supports a templating system allowing you to apply files, pkgs and
execute commands inside the containers automatically.

Currently supported template hooks are: `LIMITS`, `INCLUDE`, `PRE`, `FSTAB`,
`PKG`, `OVERLAY`, `SYSRC`, `SERVICE`, `CMD`.

Templates are created in `${bastille_prefix}/templates` and can leverage any of
the template hooks.

Bastille 0.7.x
--------------
Bastille 0.7.x introduces a template syntax that is more flexible and allows
any-order scripting. Previous versions had a hard template execution order and
instructions were spread across multiple files. The new syntax is done in a
`Bastillefile` and the template hook (see below) files are replaced with
template hook commands.

Bastille 0.8.x
--------------
Bastille 0.8.x introduces template arguments that may be defined in the 
`Bastillefile` using the ARG keyword and supplied via the command line or via an
arguments file see below for more details.

Template Automation Hooks
-------------------------

+---------+-------------------+-----------------------------------------+
| HOOK    | format            | example                                 |
+=========+===================+=========================================+
| LIMITS  | resource value    | memoryuse 1G                            |
+---------+-------------------+-----------------------------------------+
| INCLUDE | template path/URL | http?://TEMPLATE_URL or project/path    |
+---------+-------------------+-----------------------------------------+
| PRE     | /bin/sh command   | mkdir -p /usr/local/my_app/html         |
+---------+-------------------+-----------------------------------------+
| FSTAB   | fstab syntax      | /host/path container/path nullfs ro 0 0 |
+---------+-------------------+-----------------------------------------+
| PKG     | port/pkg name(s)  | vim-console zsh git-lite tree htop      |
+---------+-------------------+-----------------------------------------+
| OVERLAY | path(s)           | etc root usr (one per line)             |
+---------+-------------------+-----------------------------------------+
| SYSRC   | sysrc command(s)  | nginx_enable=YES                        |
+---------+-------------------+-----------------------------------------+
| SERVICE | service command   | 'nginx start' OR 'postfix reload'       |
+---------+-------------------+-----------------------------------------+
| CMD     | /bin/sh command   | /usr/bin/chsh -s /usr/local/bin/zsh     |
+---------+-------------------+-----------------------------------------+

Note: SYSRC requires that NO quotes be used or that quotes (`"`) be escaped
ie; (`\\"`)

Place these uppercase template hook commands into a `Bastillefile` in any order
and automate container setup as needed.

In addition to supporting template hooks, Bastille supports overlaying
files into the container. This is done by placing the files in their full path,
using the template directory as "/".

An example here may help. Think of `bastille/templates/username/template`, our
example template, as the root of our filesystem overlay. If you create an
`etc/hosts` or `etc/resolv.conf` *inside* the template directory, these
can be overlayed into your container.

Note: due to the way FreeBSD segregates user-space, the majority of your
overlayed template files will be in `usr/local`. The few general
exceptions are the `etc/hosts`, `etc/resolv.conf`, and
`etc/rc.conf.local`.

After populating `usr/local` with custom config files that your container will
use, be sure to include `usr` in the template OVERLAY definition. eg;

.. code-block:: shell

  echo "usr" > /usr/local/bastille/templates/username/template/OVERLAY

The above example "usr" will include anything under "usr" inside the template.
You do not need to list individual files. Just include the top-level directory
name. List these top-level directories one per line.

Template Arguments
------------------
You may pass variables to your `Bastillefile` via the command line or an arguments
file. Only variables in the `Bastillefile` and defined via ARG will have their 
values replaced. 

Define your arguments in the `Bastillefile` using ARG:

.. code-block:: shell

  # With a default:
  ARG user=root
  # Without a default:
  ARG domain
  # They can then by used in subsequent values:
  CMD echo "${username}@${domain}"

To pass argument values via the command line use --arg

.. code-block:: shell

  bastille template webjail nginx --arg username=admin --arg domain=example.com

To pass argument values via a file, first create a file with one definition per 
line:

.. code-block:: shell

  # values.txt
  username=admin
  domain=example.com

Then provide the path to the file using --arg-file 

.. code-block:: shell

  bastille template webjail nginx --arg-file values.txt

Template arguments will only be replaced in the `Bastillefile`, to replace any
arguments in COPY or OVERLAY files you may use the RENDER command to specify a 
file or directory whose contents should have the args replaced by their values.

.. code-block:: shell

  # etc/aliases overlay
  root: ${my_email}

  # Bastillefile
  ARG my_email
  OVERLAY etc
  RENDER /etc/aliases


Applying Templates
------------------

Containers must be running to apply templates.

Bastille includes a `template` command. This command requires a target and a
template name. As covered in the previous section, template names correspond to
directory names in the `bastille/templates` directory.

.. code-block:: shell

  ishmael ~ # bastille template ALL username/template
  [proxy01]:
  Copying files...
  Copy complete.
  Installing packages.
  pkg already bootstrapped at /usr/local/sbin/pkg
  vulnxml file up-to-date
  0 problem(s) in the installed packages found.
  Updating bastillebsd.org repository catalogue...
  [cdn] Fetching meta.txz: 100%    560 B   0.6kB/s    00:01
  [cdn] Fetching packagesite.txz: 100%  121 KiB 124.3kB/s    00:01
  Processing entries: 100%
  bastillebsd.org repository update completed. 499 packages processed.
  All repositories are up to date.
  Checking integrity... done (0 conflicting)
  The most recent version of packages are already installed
  Updating services.
  cron_flags: -J 60 -> -J 60
  sendmail_enable: NONE -> NONE
  syslogd_flags: -ss -> -ss
  Executing final command(s).
  chsh: user information updated
  Template Complete.

  [web01]:
  Copying files...
  Copy complete.
  Installing packages.
  pkg already bootstrapped at /usr/local/sbin/pkg
  vulnxml file up-to-date
  0 problem(s) in the installed packages found.
  Updating pkg.bastillebsd.org repository catalogue...
  [poudriere] Fetching meta.txz: 100%    560 B   0.6kB/s    00:01
  [poudriere] Fetching packagesite.txz: 100%  121 KiB 124.3kB/s    00:01
  Processing entries: 100%
  pkg.bastillebsd.org repository update completed. 499 packages processed.
  Updating bastillebsd.org repository catalogue...
  [poudriere] Fetching meta.txz: 100%    560 B   0.6kB/s    00:01
  [poudriere] Fetching packagesite.txz: 100%  121 KiB 124.3kB/s    00:01
  Processing entries: 100%
  bastillebsd.org repository update completed. 499 packages processed.
  All repositories are up to date.
  Checking integrity... done (0 conflicting)
  The most recent version of packages are already installed
  Updating services.
  cron_flags: -J 60 -> -J 60
  sendmail_enable: NONE -> NONE
  syslogd_flags: -ss -> -ss
  Executing final command(s).
  chsh: user information updated
  Template Complete.
