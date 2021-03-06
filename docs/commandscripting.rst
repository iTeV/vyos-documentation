.. _commandscripting:


Command scripting
=================

VyOS supports executing configuration and operational commands non-interactively from shell scripts.

To include VyOS-specific functions and aliases you need to ``source /opt/vyatta/etc/functions/script-template`` files at the top of your script.

.. code-block:: sh

  #!/bin/vbash
  source /opt/vyatta/etc/functions/script-template

  exit

Run configuration commands
--------------------------

Configuration commands are executed just like from a normal config session.

For example, if you want to disable a BGP peer on VRRP transition to backup:

.. code-block:: sh

  #!/bin/vbash
  source /opt/vyatta/etc/functions/script-template

  configure

  set protocols bgp 65536 neighbor 192.168.2.1 shutdown

  commit

  exit


Run operational commands
------------------------

Unlike a normal configuration sessions, all operational commands must be prepended with ``run``, even if you haven't created a session with configure.

.. code-block:: sh

  #!/bin/vbash
  source /opt/vyatta/etc/functions/script-template

  run show interfaces

  exit

Other script language
---------------------

If you want to script the configs in a language other than bash you can have your script output commands and then source them in a bash script.

Here is a simple example:

.. code-block:: python

  #!/usr/bin/env python
  print "delete firewall group address-group somehosts"
  print "set firewall group address-group somehosts address '1.1.1.1'"
  print "set firewall group address-group somehosts address '1.1.1.2'"


.. code-block:: sh

  #!bin/vbash
  source /opt/vyatta/etc/functions/script-template
  
  configure 
  source <(/config/scripts/setfirewallgroup.py)
  commit


Executing Configuration Scripts
-------------------------------

There is a pitfall when working with configuration scripts. It is tempting to call configuration scripts with "sudo" (i.e., temporary root permissions), because that's the common way on most Linux platforms to call system commands.

On VyOS this will cause the following problem: After modifying the configuration via script like this once, it is not possible to manually modify the config anymore:

.. code-block:: sh

  sudo ./myscript.sh # Modifies config
  configure
  set ... # Any configuration parameter

| This will result in the following error message: ``Set failed``
| If this happens, a reboot is required to be able to edit the config manually again.

To avoid these problems, the proper way is to call a script with the ``vyattacfg`` group, e.g., by using the ``sg`` (switch group) command:

.. code-block:: sh

  sg vyattacfg -c ./myscript.sh


To make sure that a script is not accidentally called without the ``vyattacfg`` group, the script can be safeguarded like this:

.. code-block:: sh

  if [ "$(id -g -n)" != 'vyattacfg' ] ; then
      exec sg vyattacfg -c "/bin/vbash $(readlink -f $0) $@"
  fi