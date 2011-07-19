Statsd setup
============

These recipes launch and configure a simple EC2 instance running Etsy's Statsd (for application statistics) and Graphite (for graphs and the web UI).


Steps
-----

If you haven't already:
  * download and install the [EC2 API tools](http://aws.amazon.com/developertools/351) (e.g., into `~/tools/`)
  * generate a X.509 certificate (both public `cert-...pem` and private `pk-...pem` files) from the [AWS Security Credentials page](https://aws-portal.amazon.com/gp/aws/developer/account/index.html?action=access-key)
  * generate a keypair (`statsd_setup.pem`) from the [AWS Management Console](https://console.aws.amazon.com)

I store the key files in `~/.ec2/` locally.

    $ sudo chmod 600 ~/.ec2/statsd_setup.pem  # set appropriate permissions on private key file

The EC2 API tools require certain environment variables to be set. I put the following in a local file (e.g., `~/bash/statsd_setup.sh`):

    export EC2_HOME=~/tools/ec2-api-tools-1.4.3.0
    export EC2_PRIVATE_KEY=~/.ec2/pk-statsd_setup.pem
    export EC2_CERT=~/.ec2/cert-statsd_setup.pem
    export JAVA_HOME=/System/Library/Frameworks/JavaVM.framework/Versions/CurrentJDK/Home
    export PATH=$PATH:$EC2_HOME/bin

...and run the following to load them:

    $ source ~/bash/statsd_setup.sh

Finally, start the instances and modify some basic security settings:

    $ ec2-run-instances ami-06ad526f -k statsd_setup  # start a new instance with a recent Ubuntu 11 image
    $ ec2-authorize default -p 22                     # permit SSH
    $ ec2-authorize default -p 80                     # permit HTTP
    $ ec2-authorize default -p 8125 -P udp            # statsd will listen here

I use [spatula](http://github.com/trotter/spatula) to apply chef recipes to simple environments. Run `ec2-describe-instances` to view the new instance's external IP, then substitute it in the following commands (run from this project's root directory):

    $ spatula prepare ubuntu@184.72.76.150 --identity ~/.ec2/statsd_setup.pem  # set up chef prerequisites
    $ spatula cook ubuntu@184.72.76.150 --identity ~/.ec2/statsd_setup.pem     # apply recipes for statsd and graphite

At this point, Statsd and Graphite are ready to start tracking metrics. Visit your instance in a browser to access Graphite's web interface.


Tracking statistics
-------------------

I added the `statsd-ruby` gem to my app's `Gemfile` and created an initializer with something like:

    require 'statsd'
    $statsd = Statsd.new('184.72.76.150')  # substitute your host

Then, tracking stats is as simple as:

    $statsd.increment('deploys')                 # increment a stat
    $statsd.increment('successful_logins', 0.1)  # increment a stat, sampling 10%
    $statsd.decrement('usage.active_users')      # decrement a stat
    $statsd.count('cart.products_added', 3)      # track an arbitrary stat
    $statsd.timing('partners.twitter_api', 650)  # track a time value (in ms)
    $statsd.time('partners.facebook_api') {...}  # track time spent executing the given block

You can always terminate the instance with `ec2-terminate-instance i-17fc3c76` (substituting the appropriate instance id).


Credits
-------

All of the included cookbooks are from third parties and can be found at the [Opscode Cookbooks Directory](http://community.opscode.com/cookbooks).