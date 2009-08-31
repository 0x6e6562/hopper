A little while ago, Rany Keddo wrote [this article][rany] about how to get [RabbitMQ][rabbit] up and running on OSX. If you are looking for a super easy way to install RabbitMQ, please read that article rather than this one. This article is just a description of how a RabbitMQ developer runs RabbitMQ on OSX.

### My Setup

I do half of my development on an [Arch Linux][arch] box and the other half on an OSX machine (10.5.5 at the moment). The way I've installed this is admittedly a little old school, but it works for me on Linux as well as OSX 10.4.

### What You'll Need

* A recent version of [XCode][], version 3 or above, in order to get a useable compiler toolchain;
* An update to date copy of [Erlang/OTP][erlang], R12B-5 or later;
* (Optionally) [Mercurial][], to get the RabbitMQ source code.

### Getting XCode

Whilst I love developing with XCode (for C projects), getting a copy can be a bit tricky. First of all, you'll need to register with the Apple developer site. Secondly, if you're not careful, the Apple site may lead you into downloading an ancient version of XCode. Have a look around to try and find out what the latest version is (I'm running 3.1.1 at the moment). Also, if installing RabbitMQ is a homework assignment, don't do this at the last minute, because the download of XCode is massive and the download speed is not the quickest.

### Installing Erlang/OTP

To install Erlang, I download the source tarball from a mirror site:

    $ curl -O http://www.csd.uu.se/ftp/mirror/erlang/download/otp_src_R12B-5.tar.gz
    
After unpacking it, I run the configure script to put the resulting Erlang installation in a local directory of my choice (because I use multiple versions of Erlang from time to time, I'd rather not bother installing it into a system wide location):
    
    $ tar zxfv otp_src_R12B-5.tar.gz
    $ cd otp_src_R12B-5
    $ ./configure --prefix=/Users/0x6e6562/Tools/R12B-5

You can obviously put this wherever you want. In the past I've had the configure script complain that library x or y wasn't present. So I installed them using macports. I think one example was ncurses, which I don't think ships with OSX. After this has run through, I do the usual:

    $ make
    $ make install
    
and now I have an Erlang interpreter in my home directory.

I do two final steps to put the _erl_ and _erl\_call_ binaries on my path:

    $ cd /Users/0x6e6562/Tools/R12B-5/bin/
    $ ln -s ../lib/erlang/lib/erl_interface-3.5.9/bin/erl_call erl_call
    $ PATH=/Users/0x6e6562/Tools/R12B-5/bin/:$PATH
    
Obviously you can just do the last step in your bash profile. After this, you should be able to just start an Erlang shell:

    $ erl
    Erlang (BEAM) emulator version 5.6.5 [source] [smp:2] [async-threads:0] [kernel-poll:false]

    Eshell V5.6.5  (abort with ^G)
    1>

Now that you have Erlang/OTP running, to get RabbitMQ up and going is a cinch.
    
### Running RabbitMQ

I have to point out that I've never bothered to install RabbitMQ because I am always developing on it. Having already installed Mercurial using the binary package from [Berkwood][], I just check out the following RabbitMQ repos:

    $ hg clone http://hg.rabbitmq.com/rabbitmq-codegen
    $ hg clone http://hg.rabbitmq.com/rabbitmq-server
    $ cd rabbitmq-server
    $ make -j run
    NODE_IP_ADDRESS= NODE_PORT= NODE_ONLY=true LOG_BASE=/tmp  RABBIT_ARGS=" -s rabbit" MNESIA_DIR=/tmp/rabbitmq-rabbit-mnesia ./scripts/rabbitmq-server
       Erlang (BEAM) emulator version 5.6.4 [source] [smp:2] [async-threads:30] [kernel-poll:true]

       Eshell V5.6.4  (abort with ^G)
       (rabbit@xlr8)1> RabbitMQ %%VERSION%% (AMQP 8-0)
       ...
       Logging to "/tmp/rabbit.log"
       SASL logging to "/tmp/rabbit-sasl.log"

       starting database             ...done
       starting core processes       ...done
       starting recovery             ...done
       starting persister            ...done
       starting builtin applications ...done
       starting TCP listeners        ...done

       broker running

And that's it, you have a running instance of RabbitMQ in an Erlang shell in foreground. If you want to, you can begin to poke around inside the shell to find out stuff, but that would be the subject of a follow up article.

### Without Mercurial

Of course, you don't have to use Mecurial to get the code, you can just download a source tarball from the RabbitMQ website and execute "make run" on the command line. This is also useful if you want to get a release version of RabbitMQ, rather than the latest version from source control.

### When Source Control Is Not The Bleeding Edge

Lastly, I have to point out that when you check out RabbitMQ from the default branch in the hg repository, whilst you are getting unreleased code, you are in actual fact _not_ getting a bleeding edge version. The reason for this lies in the fact that the development follows a branch per bug/feature methodology. This means that only when a certain piece of development has passed peer QA does it get folded into the mainline. This has advantages and disadvantages:

* Checking out the latest version of the default branch will in general give you a fairly useable version of RabbitMQ;
* If you need a bleeding edge feature, you will need to ask on the mailing list what cryptic branch name you should update your source tree once you have cloned the RabbitMQ repository.

[arch]:http://www.archlinux.org/
[rany]:http://playtype.net/past/2008/10/9/installing_rabbitmq_on_osx/
[xcode]:http://developer.apple.com/tools/xcode/
[erlang]:http://www.erlang.org
[berkwood]:http://mercurial.berkwood.com/
[mercurial]:http://www.selenic.com/mercurial/wiki/
[rabbit]:http://www.rabbitmq.com/