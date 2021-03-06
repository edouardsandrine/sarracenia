
=========
 SR_Post
=========

------------------------------------------------
Publish the Availability of a File to Subcribers
------------------------------------------------

:Manual section: 1 
:Date: @Date@
:Version: @Version@
:Manual group: MetPx Sarracenia Suite

.. contents::

SYNOPSIS
========

**sr_post|sr_cpost** [ *OPTIONS* ][ *-pb|--post_broker broker* ][ *-pbu|--post_base_url url* ] 
[ *-p|--path ] path1 path2...pathN* ]

DESCRIPTION
===========

**sr_post** posts the availability of a file by creating an announcement.
In contrast to most other sarracenia components that act as daemons,
sr_post is a one shot invocation which posts and exits.
Subscribers use `sr_subscribe <sr_subscribe.1.rst>`_  

To make files available to subscribers, **sr_post** sends the announcements 
to an AMQP server, also called a broker.  

This manual page is primarily concerned with the python implementation,
but there is also an implementation in C, which works nearly identically.
Differences:

 - plugins are not supported in the C implementation.
 - C implementation uses POSIX regular expressions, python3 grammar is slightly different.
 - when the *sleep* option ( used only in the C implementation) is set to > 0,
   it transforms sr_cpost into a daemon that works like `sr_watch(1) <sr_watch.1.rst>`_.  


Mandatory Settings
------------------

The [*-pbu|--post_base_url url*] option specifies the location 
subscribers will download the file from.  There is usually one post per file.
Format of argument to the *post_base_url* option::

       [ftp|http|sftp]://[user[:password]@]host[:port]/
       or
       file:

The [*-p|--path path1 path2 .. pathN*] option specifies the path of the files
to be announced. There is usually one post per file.
Format of argument to the *path* option::

       /absolute_path_to_the/filename
       or
       relative_path_to_the/filename

The *-pipe* option can be specified to have sr_post read path names from standard 
input as well.


An example invocation of *sr_post*::

 sr_post -pb amqp://broker.com -pbu sftp://stanley@mysftpserver.com/ -p /data/shared/products/foo 

By default, sr_post reads the file /data/shared/products/foo and calculates its checksum.
It then builds a post message, logs into broker.com as user 'guest' (default credentials)
and sends the post  to defaults vhost '/' and default exchange. The default exchange 
is the prefix *xs_* followed by the broker username, hence defaulting to 'xs_guest'.
A subscriber can download the file /data/shared/products/foo by authenticating as user stanley
on mysftpserver.com using the sftp protocol to broker.com assuming he has proper credentials.
The output of the command is as follows ::

 [INFO] Published xs_guest v02.post.data.shared.products.foo '20150813161959.854 sftp://stanley@mysftpserver.com/ /data/shared/products/foo' sum=d,82edc8eb735fd99598a1fe04541f558d parts=1,4574,1,0,0

In MetPX-Sarracenia, each post is published under a certain topic.
The log line starts with '[INFO]', followed by the **topic** of the
post. Topics in *AMQP* are fields separated by dot. The complete topic starts with
a topic_prefix (see option), version *V02*, an action *post*,
followed by a subtopic (see option) here the default, the file path separated with dots
*data.shared.products.foo*.

The second field in the log line is the message notice.  It consists of a time 
stamp *20150813161959.854*, and the source URL of the file in the last 2 fields.

The rest of the information is stored in AMQP message headers, consisting of key=value pairs.
The *sum=d,82edc8eb735fd99598a1fe04541f558d* header gives file fingerprint (or checksum
) information.  Here, *d* means md5 checksum performed on the data, and *82edc8eb735fd99598a1fe04541f558d*
is the checksum value. The *parts=1,4574,1,0,0* state that the file is available in 1 part of 4574 bytes
(the filesize.)  The remaining *1,0,0* is not used for transfers of files with only one part.

Another example::

 sr_post -pb amqp://broker.com -pbd /data/web/public_data -pbu http://dd.weather.gc.ca/ -p bulletins/alphanumeric/SACN32_CWAO_123456

By default, sr_post reads the file /data/web/public_data/bulletins/alphanumeric/SACN32_CWAO_123456
(concatenating the post_base_dir and relative path of the source url to obtain the local file path)
and calculates its checksum. It then builds a post message, logs into broker.com as user 'guest'
(default credentials) and sends the post to defaults vhost '/' and exchange 'xs_guest'.

A subscriber can download the file http://dd.weather.gc.ca/bulletins/alphanumeric/SACN32_CWAO_123456 using http
without authentication on dd.weather.gc.ca.


ARGUMENTS AND OPTIONS
=====================

Please refer to the `sr_subscribe(1) <sr_subscribe.1.rst>`_ manual page for a detailed description of 
common settings, and methods of specifying them.

[-c|--config <configfile>]
--------------------------

  A list of settings in a configuration file 

[-p|--path path1 path2 ... pathN]
---------------------------------

  **sr_post** evaluates the filesystem paths from the **path** option 
  and possibly the **base_dir** if the option is used.

  If a path defines a file, this file is announced.

  If a path defines a directory, then all files in that directory are
  announced... 

[-pb|--post_broker <broker>]
----------------------------

  the broker to which the post is sent.

[-pbd|--post_base_dir <path>]
-----------------------------

  The *base_dir* option supplies the directory path that,
  when combined (or found) in the given *path*, 
  gives the local absolute path to the data file to be posted.
  The document root part of the local path will be removed from the posted announcement.
  For sftp URLs: it can be appropriate to specify a path relative to a user account.
  Example of that usage would be:  -dr ~user  -post_base_url sftp:user@host  
  For file URLs: base_dir is usually not appropriate.  To post an absolute path, 
  omit the -dr setting, and just specify the complete path as an argument.

[-px|--post_exchange <exchange>]
--------------------------------

  Sr_post publishes to an exchange named *xs_*"broker_username" by default.
  Use the *post_exchange* option to override that default.

[-h|-help|--help]
-----------------

  Display program options.

[--blocksize <value>]
---------------------

  This option controls the partitioning strategy used to post files.
  The value should be one of::

     0 - autocompute an appropriate partitioning strategy (default)
     1 - always send entire files in a single part.
     <blocksize> - used a fixed partition size (example size: 1M )

  Files can be announced as multiple parts.  Each part has a separate checksum.
  The parts and their checksums are stored in the cache. Partitions can traverse
  the network separately, and in parallel.  When files change, transfers are
  optimized by only sending parts which have changed.  
  
  The value of the *blocksize*  is an integer that may be followed by  letter designator *[B|K|M|G|T]* meaning:
  for Bytes, Kilobytes, Megabytes, Gigabytes, Terabytes respectively.  All these references are powers of 2.
  Files bigger than this value will get announced with *blocksize* sized parts.
  
  The autocomputation algorithm determines a blocksize that encourages a reasonable number of parts
  for files of various sizes.  As the file size varies, the automatic computation will give different
  results.  This will result in resending information which has not changed as partitions of a different 
  size will have different sums, and therefore be tagged as different.  
  
  By default, **sr_post** computes a reasonable blocksize that depends on the file size.
  The user can set a fixed *blocksize* if it is better for its products or if he wants to
  take advantage of the **cache** mechanism.  In cases where large files are being appended to, for example,
  it make sense to specify a fixed partition size so that the blocks in the cache will be the 
  same blocks as those generated when the file is larger, and so avoid re-transmission.  So use 
  of '10M' would make sense in that case.  
  
  In cases where a custom downloader is used which does not understand partitioning, it is necessary
  to avoid having the file split into parts, so one would specify '1' to force all files to be sent
  as a single part.

[-pbu|--post_base_url <url>]
----------------------------

  The **url** option sets the protocol, credentials, host and port under
  which the product can be fetched.

  The AMQP announcememet is made of the three fields, the announcement time,
  this **url** value and the given **path** to which was withdrawn from the *base_dir*
  if necessary.

  The concatenation of the two last fields of the announcement defines
  what the subscribers will use to download the product. 

[-pipe <boolean>]
-----------------

  The pipe option is for sr_post to read the names of the files to post from standard input to read from
  redirected files, or piped output of other commands. Default is False, accepting file names only on the command line.

[--pulse_message <message>]
---------------------------

  Administrator option to send a message to all subscribers.  Similar to "wall" functionality.
  When this option is set, a pulse message is sent, ignoring any topic settings or files given as arguments.

[--reset]
---------

  When one has used **--suppress_duplicates|--cache**, this option empties the cache.


[-rn|--rename <path>]
---------------------

  With the *rename*  option, the user can suggest a destination path to its files. If the given
  path ends with '/' it suggests a directory path...  If it doesn't, the option specifies a file renaming.

[--sleep <time> ]
-----------------

   **This option is only available in the c implementation (sr_cpost)**

   When the option is set, it transforms cpost into a sr_watch, with *sleep* being the time to wait between 
   generating events.  When files are written frequently, it is counter productive to produce a post for 
   every change, as it can produce a continuous stream of changes where the transfers cannot be done quickly 
   enough to keep up.  In such circumstances, one can group all changes made to a file
   in *sleep* time, and produce a single post.

   NOTE::
       in sr_cpost, when combined with force_polling (see `sr_watch(1) <sr_watch.1.rst>`_ ) the sleep 
       interval should not be less than about five seconds, as it may miss posting some files.

   

[-sub|--subtopic <key>]
-----------------------

  The subtopic default can be overwritten with the *subtopic* option.


[--suppress_duplicates|-sd|-nd|--no_duplicates|--cache on|off|999]
------------------------------------------------------------------

  Avoid posting duplicates. When posting directories, this option caches
  what was posted and will post only files (or parts of files) that were new
  when invoked again. 
 
  Over time the number of files in the cache can grow too large, and so it is cleaned out of
  old entries.  The default lifetime of a cache entry is five minutes (300 seconds). This
  lifetime can be overridden with a time interval as argument ( the 999 above ).

  If duplicate suppression is in use,  one should ensure that a fixed **blocksize** is
  used ( set to a value other than 0 ) as otherwise blocksize will vary as files grow,
  and much duplicate data transfer will result.

[-to|--to <destination>,<destination>,... ]
-------------------------------------------

  A comma-separated list of destination clusters to which the posted data should be sent.
  Ask pump administrators for a list of valid destinations.

  Default: the hostname of the broker.

  *FIXME: a good list of destination should be discoverable.*

[-sum|--sum <string>]
---------------------

  All file posts include a checksum.  The *sum* option specifies how to calculate the it.
  It is a comma separated string.  Valid checksum flags are ::

    [0|n|d|s|N|z]
    where 0 : no checksum... value in post is a random integer (only for testing/debugging.)
          d : do md5sum on file content (default for now, compatibility)
          n : do md5sum checksum on filename
          N : do SHA512 checksum on filename
          s : do SHA512 on file content (default in future)
          z,a : calculate checksum value using algorithm a and assign after download.

  Then using a checksum script, it must be registered with the pumping network, so that consumers
  of the postings have access to the algorithm.


[-tp|--topic_prefix <key>]
--------------------------

  *Not usually used*
  By default, the topic is made of the default topic_prefix : version *V02*, an action *post*,
  followed by the default subtopic: the file path separated with dots (dot being the topic separator for amqp).
  You can overwrite the topic_prefix by setting this option.



[-header <name>=<value>]
------------------------

  Add a <name> header with the given value to advertisements. Used to pass strings as metadata.




ADMINISTRATOR SPECIFIC
======================

[--queue|--queue_name|-qn] <queue>
----------------------------------

  If a client wants a product to be reannounced,
  the broker administrator can use *sr_post*  and publish
  directly into the client's queue. The client could provide
  his queue_name... or the administrator would find it on
  the broker... From the log where the product was processed on
  the broker, the administrator would find all the messages
  properties. The administrator should pay attention on slight
  differences between the logs properties and the *sr_post* arguments.
  The logs would mention *from_cluster*  *to_clusters* and associated
  values...  **sr_post** arguments would be *-cluster* and  *-to*
  respectively. The administrator would execute **sr_post**, providing
  all the options and setting everything found in the log plus 
  *-queue_name q_....*



DEVELOPER SPECIFIC OPTIONS
==========================

[-debug|--debug]
----------------

  put more messages in the log.

[-r|--randomize]
----------------

  If a file is posted in several blocks, the posting order
  is randomized so that the subcriber receives them out of order.

[-rc|--reconnect]
-----------------

  If a file is posted in several blocks, reconnect to the broker
  for every post. 

[--parts]
---------

  The usual usage of the *blocksize* option is described above, which is what is usually used to set
  the *parts* header in the messages produced, however there are a number of ways of using the parts flag 
  that are not generally useful aside from within development.
  In addition to the user oriented *blocksize* specifications listed before, any valid 'parts' header, as given in the 
  parts header (e.g. 'i,1,150,0,0') .  One can also specify an alternate basic blocksize for the automatic 
  algorithm by giving it after the '0', (eg. '0,5') will use 5 bytes (instead of 50M) as the basic block size, so one
  can see how the algorithm works.


ENVIRONMENT VARIABLES
=====================

In the C implementation (sr_cpost), if the SR_CONFIG_EXAMPLES variable is set, then the *add* directive can be used
to copy examples into the user's directory for use and/or customization.

An entry in the ~/.config/sarra/default.conf (created via sr_subscribe edit default.conf )
could be used to set the variable::

  declare env SR_CONFIG_EXAMPLES=/usr/lib/python3/dist-packages/sarra/examples

the value should be available from the output of a list command from the python
implementation.



SEE ALSO
========

`sr_report(7) <sr_report.7.rst>`_ - the format of report messages.

`sr_report(1) <sr_report.1.rst>`_ - process report messages.

`sr_post(7) <sr_post.7.rst>`_ - the format of announcement messages.

`sr_sarra(8) <sr_sarra.8.rst>`_ - Subscribe, Acquire, and ReAdvertise tool.

`sr_subscribe(1) <sr_subscribe.1.rst>`_ - the http-only download client.

`sr_watch(1) <sr_watch.1.rst>`_ - the directory watching daemon.



