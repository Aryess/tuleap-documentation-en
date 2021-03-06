Development set-up
==================

Virtual Machine
---------------
This document explains the various steps needed to have a tuleap server on a virtual machine. This box will be for development purpose, thus it is assumed that you already have cloned the sources of Tuleap in a directory on the host (e.g: $HOME/tuleap/).

Docker
``````
Introduction
"""""""""""""


Prerequisites:

- docker
- JQ

Start a new docker container
""""""""""""""""""""""""""""
  .. code-block:: bash

    $ cd /path/to/tuleap_workspace
    $ tools/docker/dev_start.sh <name_of_the_server>

And voila, your server is up and running. The first time you run this command, docker will download tuleap base image. It's 1,6GB so please be patient.
Edit the /etc/hosts file of your host and add the line given by the script. Something like:

  .. code-block:: bash

    172.17.0.4     <name_of_the_server>.<name_of_host>

You can start as many servers as you like, but they will all share the current tuleap source.
  
  .. NOTE:: Please note that the docker image is read-only, and every modification to the OS will be lost at reboot. 
   If you need to add/change anything and make it persistant, fork and ammend the Dockerfile (https://registry.hub.docker.com/u/enalean/tuleap-aio-dev/)
   Everything but the OS (tuleap config, database, user home) is saved in /srv/dock/<name_of_the_server> on the host.
   

Advanced setup
""""""""""""""

- This will start a Tuleap image named 'tuleap', and link it to a Elastic Search image named 'elast'

    .. code-block:: bash
    
      docker run -d --name=elast enalean/elasticsearch-tuleap
      docker run -d --name=tuleap --link elast enalean/tuleap-aio-dev

- You can add a LDAP server for development purpose with:
    .. code-block:: bash

      $ docker run -p 389:389 -d /srv/docker/ldap:/data enalean/ldap-dev
    
  Then you can start adding people (you can find a template here: https://github.com/Enalean/docker-ldap-dev/blob/master/bob.ldif):

  .. code-block:: bash

      $ ldapadd -f bob.ldif -x -D 'cn=Manager,dc=tuleap,dc=local' -w welcome0
      $ ldapsearch -x -LLL -b 'dc=tuleap,dc=local' 'cn=bob*'
    
  Notes:
    * The IP address you need to declare in Tuleap ldap plugin is the one of your host machine
    * You might also want to use --link docker option instead of publish 389 on your localhost
    * /srv/docker/ldap is were data will be stored on your host
    
    
Vagrant
```````
Introduction
"""""""""""""


Prerequisites:

- vagrant greater or equals to 1.4.1
- virtualbox greater of equals to 4.3.6

Start the vagrant box
"""""""""""""""""""""

  .. code-block:: bash

    $ cd /path/to/tuleap_workspace
    $ git clone gitolite@tuleap.net:tuleap/tuleap/stable.git tuleap
    $ git clone gitolite@tuleap.net:tuleap/tools/vagrant.git vagrant
    $ cd vagrant
    $ git submodule init
    $ git submodule update
    $ vagrant up

Edit the /etc/hosts file of your host and add the following line:

  .. code-block:: bash

    10.11.13.11    tuleap.local

You can now access Tuleap in your browser with the following url: http://tuleap.local/

You can start coding with your prefered IDE (we recommend netbeans) on your local machine.

Manual setup
````````````

Sharing files with host with nfs
"""""""""""""""""""""""""""""""""

Virtual box shared folder are far too slow to be used without being mad after a couple of minutes.
So you can use NFS to share stuff between your host and your guest (for instance eclipse workspace if you use it).

In Virtual Box configuration:

- Setup a second interface (the first one was NATed) with "Host-only adaptater" and "vboxnet0"
- Then you should have a new interface on your host:


    .. code-block:: bash

        $> ifconfig -a
        vboxnet0  Link encap:Ethernet  HWaddr 0a:00:27:00:00:00
                  inet addr:192.168.56.1  Bcast:192.168.56.255  Mask:255.255.255.0
                  inet6 addr: fe80::800:27ff:fe00:0/64 Scope:Link
                  UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
                  RX packets:0 errors:0 dropped:0 overruns:0 frame:0
                  TX packets:80 errors:0 dropped:0 overruns:0 carrier:0
                  collisions:0 txqueuelen:1000
                  RX bytes:0 (0.0 B)  TX bytes:16188 (16.1 KB)

If you boot the VM, the guest now have a new interface as well:

    .. code-block:: bash

        $> ifconfig -a
        eth1  Link encap:Ethernet  HWaddr 08:00:27:51:EA:5C
              inet addr:192.168.56.101  Bcast:192.168.56.255  Mask:255.255.255.0
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              RX packets:19716 errors:0 dropped:0 overruns:0 frame:0
              TX packets:19001 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:1000
              RX bytes:6713350 (6.4 MiB)  TX bytes:3892833 (3.7 MiB)

For HTTPD to work with an NFS-mounted document root, you will probably need to disable SELinux on the guest:

    Edit /etc/selinux/config, and change the following line:

    .. code-block:: bash

        SELINUX=disabled

If you don't want to reboot for your changes to be applied, use the following command:

    .. code-block:: bash

        setenforce 0

On the host: setup nfs server:

- Install the required packages (Ubuntu: sudo apt-get install nfs-kernel-server ; nothing to do on MacOS X)
- Create a new directory for your user sudo mkdir /srv/myname and change permissions: sudo chown myname:myname /srv/myname
- Edit /etc/exports (on Linux):

    .. code-block:: bash

        /srv/myname        192.168.56.101(rw,sync,no_subtree_check,anonuid=1000,anongid=1000,all_squash)

Or on MacOS X :

    .. code-block:: bash

        /Users/sebn/Projets/tuleap -alldirs -mapall=yourusername

Restart nfs (on Linux):

    .. code-block:: bash

        sudo /etc/init.d/nfs-kernel-server restart

Or on MacOS X:

    .. code-block:: bash

        sudo nfsd restart

Notes:

    ip address is the one of VM "host-only" interface (eth1 in our example)
    anonuid & anongid refer to the numerical id of your user on the host (myname) you can get them with (id myname)
    anonuid=1000,anongid=1000,all_squash will force all writes on the VM to be remaped as your username on host.

On the guest: setup the nfs client:

You can test with mount 192.168.56.1:/srv/myname /mnt (please note the ip address, the IP associated to vboxnet0 on host).
If it works, unmount (umount /mnt) it and add to /etc/fstab:

    .. code-block:: bash

        192.168.56.1:/srv/myname /mnt/myname    nfs     rw,auto,rw 0 0

And mount it mount /mnt/myname

Now you are ready to use your host files on the client. If you already have a workspace on your homedir, you should move it into the shared area (mv /workspace /srv/myname).

And finally, replace the existing Tuleap install by the development one:

    .. code-block:: bash

        $> cd /usr/share
        $> mv codendi codendi_rpm
        $> ln -s /mnt/myname/workspace/Tuleap codendi
        $> service httpd restart

Sharing files with host with lxc
"""""""""""""""""""""""""""""""""

Note : do these steps on PHP 5.1 lxc virtual machines before run the setup.sh

In the /var/lib/lxc/myLxcVirtualName/config add the line

    .. code-block:: bash

        lxc.mount.entry=/srv/myTuleapDir /var/lib/lxc/myLxcVirtualName/rootfs/mnt none bind  0 0

In order to let your lxc host access the /mnt, get the uid and gid by using the 'll' command, and the original uid and gid by 'id codendiadm' Then:

    .. code-block:: bash

        usermod -u youruid codendiadm
        groupmod -g yourgid codendiadm
        find / -uid yourolduid -exec chown codendiadm {} \;
        find / -gid youroldgid -exec chgrp codendiadm {} \;
        reboot

Git workflow
------------

Development repository is hosted on http://gerrit.tuleap.net

You can checkout either from ssh or http: http://gerrit.tuleap.net/#/admin/projects/tuleap

Alternative repositories
````````````````````````

The reference repository, stable, is "the true master" (ie. it's from this repository that releases are rolled out).

There are mirrors of stable repository available:

- On Github public/anonymous access. Synchronized on every push on master.

Setting up your environment
```````````````````````````

1. configure your local config to rebase when you pull changes locally:

  .. code-block:: bash

    $> git config branch.autosetuprebase always

2. install local hooks:

  .. code-block:: bash

    $> cp .git/hooks/pre-commit.sample .git/hooks/pre-commit
    $> curl -o .git/hooks/commit-msg http://gerrit.tuleap.net/tools/hooks/commit-msg
    $> chmod u+x .git/hooks/commit-msg

3. Configure your gerrit environement

Setup you account (please use the same login name than on tuleap.net) on http://gerrit.tuleap.net and publish your ssh key (not needed if you are using http as transport).

  .. code-block:: bash

    $> git remote add gerrit ssh://USERNAME@gerrit.tuleap.net:29418/tuleap.git

LESS
-----

What's LESS ?
``````````````

LESS files are just extended CSS files. It means you can use variables, functions, operations and more in CSS files very easily. It's fully backward compatible with exiting CSS files (you can rename file.css to file.less, compile file.less and it'll just work).

Please refer to the LESS documentation for more information.

Install Recess in Tuleap environment
`````````````````````````````````````

Download and install NodeJS if needed
""""""""""""""""""""""""""""""""""""""

Download the NodeJS binaries here.

Put the archive wherever you want and extract it:

    .. code-block:: bash

        mv node-v0.10.21-linux-x64.tar.gz /usr/local/src
        cd /usr/local/src
        tar -zxvf node-v0.10.21-linux-x64.tar.gz
        ln -s node-v0.10.21-linux-x64 node

You have to add NodeJS to your path. To do so, edit your profile file. For example, if you use bash:

    .. code-block:: bash

        vi ~/.bash_profile

Add or edit the line containing your PATH definition:

    .. code-block:: bash

        export PATH=$PATH:/usr/local/src/node/bin

Then, if necessary, source your console's profile:

    .. code-block:: bash

        source ~/.bash_profile

Download and install Less using npm if needed
""""""""""""""""""""""""""""""""""""""""""""""

Run this command:

    .. code-block:: bash

        npm install less -g

Check that everything went fine:

    .. code-block:: bash

        lessc -v

Download and install Recess using npm if needed
""""""""""""""""""""""""""""""""""""""""""""""""

Run this command:

    .. code-block:: bash

        npm install recess -g

Check that everything went fine:

    .. code-block:: bash

        recess -v

Compile LESS files
```````````````````

You are now able to compile LESS files. Just go to your tuleap installation directory:

    .. code-block:: bash

        cd /usr/share/codendi

And compile LESS files:

    .. code-block:: bash

        make less

This command will compile all LESS files present in plugin and src directories. One CSS file will be created/updated for each LESS file.

Keep in mind that:

- you have to run make less everytime you edit a LESS file except if you have enabled the dev mode.
- all modifications must be done in LESS file, not in CSS file.

Use the development mode
`````````````````````````

Add EPEL repos if needed
"""""""""""""""""""""""""

    .. code-block:: bash

        wget http://dl.fedoraproject.org/pub/epel/5/x86_64/epel-release-5-4.noarch.rpm
        wget http://rpms.famillecollet.com/enterprise/remi-release-5.rpm
        rpm -Uvh remi-release-5*.rpm epel-release-5*.rpm

Download inotify-tools if needed
""""""""""""""""""""""""""""""""

    .. code-block:: bash

        yum install inotify-tools

Launch the development mode
""""""""""""""""""""""""""""

Launch make less-dev to watch modifications on LESS files. Everytime a LESS file is modified, it will be recompiled automatically.

Just go to your tuleap installation directory:

    .. code-block:: bash

        cd /usr/share/codendi

And launch the development mode:

    .. code-block:: bash

        make less-dev

Use Ctrl+C to quit the development mode

FAQ
````

OMG, there are barely understandable error while compiling less files
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

    .. code-block:: bash

        [tuleap] make less
        […]
        Compiling /home/nicolas/tuleap/src/www/themes/KASS/css/style.less

        /usr/local/lib/node_modules/recess/node_modules/less/lib/less/parser.js:421
                                throw new(LessError)(e, env);
                                      ^
        [object Object]
        […]

To have more details about the error you can issue the lessc command on the incriminated file:

    .. code-block:: bash

        [tuleap] lessc /home/nicolas/tuleap/src/www/themes/KASS/css/style.less
        NameError: variable @inputHeight is undefined in /home/nicolas/tuleap/src/www/themes/common/css/bootstrap-2.3.2/mixins.less on line 157, column 15:
        156   width: 100%;
        157   min-height: @inputHeight; // Make inputs at least the height of their button counterpart (base line-height + padding + border)
        158   .box-sizing(border-box); // Makes inputs behave like true block-level elements

LDAP
----

Setup
``````

    .. code-block:: bash

        yum install openldap-servers

Tests
------

We strongly encourage developers to apply TDD. Not only as a test tool but as a design tool.

Run tests
``````````

Tuleap comes with a handy test environment, based on SimpleTest. File organization:

- Core tests (for things in src directory) can be found in tests/simpletest directory with same subdirectory organization (eg. src/common/frs/FRSPackage.class.php tests are in tests/simpletest/common/frs/FRSPackageTest.php).
- Plugins tests are in each plugin tests directory (eg. plugins/tracker/include/Tracker.class.php tests are in plugins/tracker/tests/TrackerTest.php).

To run tests you can either use:

- the web interface available at http://localhost/plugins/tests/ (given localhost is your development server)
- the CLI interface: make tests (at the root of the sources). You can run a file or a directory: php tests/bin/simpletest plugins/docman

Integration tests
"""""""""""""""""

A couple of tests interact with the database to ensure whole stack consistency.

You cannot run them from the web interface yet, you should run it by hand:

    .. code-block:: bash

        $> php tests/bin/simpletest tests/integration plugins/tracker/db_tests

For this to work, you need to create a database for tests in your development environment (as mysql root):

    .. code-block:: bash

        mysql> GRANT ALL PRIVILEGES on integration_test.* to 'integration_test'@'localhost' identified by 'welcome0';

Organize your tests
````````````````````

All the tests related to one class (therefore to one file) should be kept in one test file (src/common/foo/Bar.class.php tests should be in tests/simpletest/common/foo/BarTest?.php). However, we strongly encourage you to split test cases in several classes to leverage on setUp.

    .. code-block:: bash

        class Bar_IsAvailableTest extends TuleapTestCase {
            //...
        }

        class Bar_ComputeDistanceTest extends TuleapTestCase {
            //...
        }

Of course, it's by no mean mandatory and always up to the developer to judge if it's relevant or not to split tests in several classes. A good indicator would be that you can factorize most of tests set up in the setUp method. But if the setUp contains things that are only used by some tests, it's probably a sign that those tests (and corresponding methods) should be in a dedicated class.

Write a test
````````````

What makes a good test:

- It's simple
- It has an explicit name that fully describes what is tested
- It tests only ONE thing at a time

Diffrences with simpletest:

- tests methods can start with it keyword instead of test. Example: public function itThrowsAnExceptionWhenCalledWithNull()

On top of simpletest we added a bit of syntactic sugar to help writing readable tests. Most of those helpers are meant to help dealing with mock objects.

class Bar_IsAvailableTest extends TuleapTestCase {

   .. code-block:: bash

            public function itThrowsAnExceptionWhenCalledWithNull() {
                $this->expectException();
                $bar = new Bar();
                $bar->isAvailable(null);
            }

            public function itIsAvailableIfItHasMoreThan3Elements() {
                $foo = mock('Foo');
                stub($foo)->count()->returns(4);
                //Syntaxic sugar for :
                //$foo = new MockFoo();
                //$foo->setReturnValue('count', 4);

                $bar = new Bar();
                $this->assertTrue($bar->isAvailable($foo));
            }

            public function itIsNotAvailableIfItHasLessThan3Elements() {
                $foo = stub('Foo')->count()->returns(2);

                $bar = new Bar();
                $this->assertFalse($bar->isAvailable($foo));
            }
}

Available syntaxic sugars SimpleTest:
   .. code-block:: bash

            Mock::generate('Foo'); $foo = new MockFoo();
            $foo->setReturnValue('bar', 123, array($arg1, $arg2));
            $foo->expectOnce('bar', array($arg1, $arg2));
            $foo->expectNever('bar');
            $foo->expectAt(2, 'bar', array($arg1, $arg2));
            $foo->expectCallCount('bar', 4);

Tuleap:
   .. code-block:: bash

            $foo = mock('Foo');
            stub($foo)->bar($arg1, $arg2)->returns(123);
            stub($foo)->bar($arg1, $arg2)->once();
            stub($foo)->bar()->never();
            stub($foo)->bar(arg1, arg2)->at(2);
            stub($foo)->bar()->count(4);


See details and more helpers in plugins/tests/www/MockBuilder.php.

Helpers and database
`````````````````````

A bit of vocabulary:

    Interactions between Tuleap and the database should be done via DataAccessObject (aka. dao) objects (see src/common/dao/include/DataAccessObject.class.php)
    A dao that returns rows from database wrap the result in a DataAccessResult (aka. dar) object (see src/common/dao/include/DataAccessResult.class.php)

Tuleap test helpers ease interaction with database objects. If you need to interact with a query result you can use mock's returnsDar, returnsEmptyDar and returnsDarWithErrors.

   .. code-block:: bash

            public function itDemonstrateHowToUseReturnsDar() {

                $project_id = 15;
                $project    = stub('Project')->getId()->returns($project_id);

                $dao        = stub('FooBarDao')->searchByProjectId($project_id)->returnsDar(
                    array(
                        'id'  => 1
                        'name' => 'foo'
                    ),
                    array(
                        'id'  => 2
                        'name' => 'klong'
                    ),
                );

                $some_factory = new Some_Factory($dao);
                $some_stuff   = $some_factory->getByProject($project);
                $this->assertEqual($some_stuff[0]->getId(), 1);
                $this->assertEqual($some_stuff[1]->getId(), 2);
            }

Builders
`````````

Keep tests clean, small and readable is a key for maintainability (and avoid writing crappy tests). A convenient way to simplify tests is to use Builder Pattern to wrap build of complex objects.

Note: this is not an alternative to partial mocks and should be used only on "Data" objects (logic less, transport objects). It's not a good idea to create a builder for a factory or a manager.

At time of writing, there are 2 builders in Core aUser.php and aRequest.php:

   .. code-block:: bash

            public function itDemonstrateHowToUseUserAndRequest() {

                $current_user = aUser()->withId(12)->withUserName('John Doe')->build();
                $new_user     = aUser()->withId(655957)->withUserName('Usain Bolt')->build();
                $request      = aRequest()->withUser($current_user)->withParam('func', 'add_user')->withParam('user_id', 655957)->build();

                $some_manager = new Some_Manager($request);
                $some_manager->createAllNewUsers();
            }

There are plenty of builders in plugins/tracker/tests/builders and you are strongly encouraged to add new one when relevant.
