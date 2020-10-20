# dokuwiki-sync

Shell script to sync remote DokuWiki instance to local one through XMLRPC using [dokuJClient](https://github.com/gturri/dokuJClient).

Please note that ``dokuwiki-sync`` is a unidirectional script: it doesn't sync (yet, at least) two DokuWiki instances, it only overwrites local content with remote one if local content is different than the remote one; even if local content is newer than remote content, it will be overwritten by remote content. So you'll need to make sure that you edit common pages between DokuWiki instances into remote wiki instead of local one, or your changes will be overwritten during ``dokuwiki-sync`` execution.

# Screenshots

``![dokuwiki-sync syncing pages from remote wiki to local wiki](/screenshots/dokuwiki-sync-pages.png)``
 
``dokuwiki-sync`` script syncing pages from remote wiki to local wiki.

``![dokuwiki-sync syncing attachments from remote wiki to local wiki](/screenshots/dokuwiki-sync-attachments.png)``
 
``dokuwiki-sync`` script syncing attachments from remote wiki to local wiki.

Note that in both cases, ``dokuwiki-sync`` script compares the remote metadata with local metadata and only download/overwrite the local wiki file if local metadata is different than remote one.
 
# Getting Started

## Installing dokuJClient

It may be installed from the packages on several Linux distributions:

    sudo apt-get install dokujclient
    
If it isn't available in the packages of your plataform you may:

1. Download the [binaries](http://turri.fr/dokujclient).

    curl -L -o /tmp/dokujclient-3.9.1-bin.tar.gz http://turri.fr/dokujclient/dokujclient-3.9.1-bin.tar.gz
   
2. Extract it, and add the extracted directoy to your path.

    mkdir /usr/local/dokujclient && tar -zxf /tmp/dokujclient-3.9.1-bin.tar.gz --strip-components=1 -C /usr/local/dokujclient
    chown -Rf root:root /usr/local/dokujclient
    ln -s /usr/local/dokujclient/dokujclient /usr/local/bin
   
3. Ensure it's correctly installed, typing e.g.:

    dokujclient --version

## Getting the dokuwiki-sync script

    curl -L -o /usr/local/bin/dokuwiki-sync https://raw.githubusercontent.com/coldscientist/dokuwiki-sync/main/dokuwiki-sync
    chmod +x /usr/local/bin/dokuwiki-sync

# Parameters

Usage: dokuwiki-sync [-c|--config-file CONFIG_FILE] [-a|--attachments-only] [-p|--pages-only] [-d|--debug]        

Parameter | Description
--------- | -----------
``-h, --help`` | Show this help message and exit.
``-c, --config-file CONFIG_FILE`` | Load CONFIG_FILE configuration.
``-a, --attachments-only`` | Sync attachments only.
``-p, --pages-only`` | Sync remotepages only.
``-d, --debug`` | Print debug information.

## Config file (``.dokujclientrc``)

You'll need to specify the url, user, and password to dokuJClient connect to the remote DokuWiki instance. You can create a ``.dokujclientrc`` file and put some or all of this info in it.

    echo "url=http://myhost/mywiki/lib/exe/xmlrpc.php" > ~/.dokujclientrc
    echo "user=toto" >> ~/.dokujclientrc
    echo "password=myPassword" >> ~/.dokujclientrc

By default, the ``dokuwiki-sync`` script attempts to load dokuJClient configuration parameters (``url``, ``user`` and ``password`` of remote DokuWiki instance) into the following sequence:

1. ``$HOME/.dokujclientrc`` config file.
2. ``$DWCONFIGDIRNAME/.dokujclientrc``, where ``$DWCONFIGDIRNAME`` is the directory path that ``.dokuwiki-syncrc`` file resides.
3. ``$HOME/private/.dokujclientrc``
4. ``$HOME/etc/.dokujclientrc``
5. ``"$(pwd)/.dokujclientrc``, where ``$(pwd)`` is the current path that the ``dokuwiki-sync`` script was executed.
6. ``.dokuwiki-syncrc`` config file.
7. Environment variables (``url``, ``user`` and ``password``).
8. Fails.

## Config file (``.dokuwiki-syncrc``)

By default, the ``dokuwiki-sync`` script attempts to load it's configuration parameters into the following sequence:

1. Through ``-c | --config-file`` parameter, e.g. ``dokuwiki-sync --config-file [web_root]/../private/.dokuwiki-syncrc``
2. ``$HOME/.dokuwiki-syncrc`` config file.
3. ``$HOME/private/.dokuwiki-syncrc``
4. ``$HOME/etc/.dokuwiki-syncrc``
5. ``"$(pwd)/.dokujclientrc``, where ``$(pwd)`` is the current path that the ``dokuwiki-sync`` script was executed.
6. Parameters are loaded from Environment variables.
7. The ``dokuwiki-sync`` script checks common file paths and guess the parameters.
8. Fails.

A valid ``.dokuwiki-syncrc`` configuration file would be:

    localwikidir=/var/www/clients/client0/web3/web
    localwikisavedir=$localwikidir/data
    localwikidatadir=$localwikidatadir/pages
    localwikimediadir=$localwikidatadir/media
    localwikichown=5006
    localwikichgrp=5005

Parameter | Optional |Description
--------- | --------- |-----------
``localwikidir`` | Yes. | Specifies the local DokuWiki directory, e.g. ``/var/www/html/dokuwiki``
``localwikisavedir`` | Yes. | Specifies the ``data/pages`` default directory for pages sync. Default: ``$localwikidir/data/pages``
``localwikidatadir`` | Yes. | Specifies the ``data/media`` default directory for attachments sync. Default: ``$localwikidir/data/media``
``localwikichown`` | Yes. | Specifies the UID (e.g. ``0``) or Username (e.g. ``root``) of the user that the directory ``$localwikisavedir/meta`` belongs (more details below).
``localwikichgrp`` | Yes. | Specified the GID (e.g. ``0``) or Groupname of the group that the directory ``$localwikisavedir/meta`` belongs (more details below).

**Notes about ``localwikichown`` and ``localwikichgrp``**

The ``localwikichown`` and ``localwikichgrp`` parameters are useful if you're running this script as root, because it allows you to change the owner of pages and attachment files using UID or username. E.g. if you deployed this script at ``/etc/cron.daily``, the script will run with the user ``root`` by default, but the ``apache``/``httpd`` daemon runs using another user than ``root`` by default and the daemon would not be able to access the pages and attachments saved by ``root``. So these parameters allows you to change the user and group that the directory ``$localwikisavedir/meta`` belongs so the permissions are correctly applied to the user and group specified.

By default, the script will attempts to parse the current user and group that the directory ``$localwikisavedir/meta`` belongs and will try to change the user and group according **IF** the parameters ``localwikichown`` and ``localwikichgrp`` are not set.

# Purpose

I tried to sync a local DokuWiki instance using the [Sync](https://www.dokuwiki.org/plugin:sync) DokuWiki plugin, but for some reason, I was receiving the error ``transport error - Could not connect to ssl://xxxxx (0)`` when trying to connect to my remote DokuWiki instance. At first, I thought that it could be related to "remote" and "remoteuser" entries in my remote DokuWiki configuration or something related to my remote DokuWiki instance certificated be emitted by Let's Encrypt CA (maybe the local DokuWiki instance wasn't trusting it), but I successfully connected to my remote DokuWiki instance using xcom DokuWiki plugin, so I was convicted that the error wasn't related to my setup or XMLRPC issues but with the [Sync](https://www.dokuwiki.org/plugin:xcom) plugin itself.

I decided to migrate to MediaWiki and use the [CloneDiff](https://www.mediawiki.org/wiki/Extension:CloneDiff) MediaWiki extension, but I noticed that the learning curve of MediaWiki is higher than DokuWiki and there isn't a easy replacement to [RefNotes](https://www.dokuwiki.org/plugin:refnotes) DokuWiki plugin and others. So I decided to stick with DokuWiki, but I needed to solve this issue and find a way to sync a remote DokuWiki instance with a local one.

Initially, I thought using a FTP-like solution to sync my remote DokuWiki instance with a local one or use the [gitbacked](https://www.dokuwiki.org/plugin:gitbacked) DokuWiki plugin, but I didn't wanted to sync all pages. I would need to: 1) keep a local exclusion list of attachments and pages that I wouldn't like to sync OR 2) defining the pages that I would like to sync manually OR 3) syncing all files and create an exclusion list that would delete all undesired synced pages and attachments namespaces (bandwidth waste...). It would not be a viable approach.

The XMLRPC interface that DokuWiki exposes allow defining an ACL (Access Control List) into remote DokuWiki instance that would be respected by the local DokuWiki instance, e.g. you only need to enable the ``remote`` config and add the user or group that the user belongs to ``remoteuser`` config at remote DokuWiki instance and when connecting through XMLRPC with the specified ``remoteuser``, it would inherit the ACL permissions defined at the remote Access Control List configuration, so I could enable and disable anytime namespaces for synchronization using the remote DokuWiki admin page.

As XMLRPC interface was working through xcom DokuWiki plugin, I decided to find a way to use this protocol to sync my local DokuWiki instance with the remote one, but I couldn't find any easy-to-use solution to sync a local DokuWiki instance with a remote one, so when I find [dokuJClient](https://github.com/gturri/dokuJClient) project, I decided to develop one using it.

# License

[GNU General Public License v3.0](LICENSE) License.
