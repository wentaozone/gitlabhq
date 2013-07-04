# Select Version to Install
Make sure you view this installation guide from the branch (version) of GitLab you would like to install. In most cases
this should be the highest numbered stable branch (example shown below). 

![capture](https://f.cloud.github.com/assets/1192780/564911/2f9f3e1e-c5b7-11e2-9f89-98e527d1adec.png)

If this is unclear check the [GitLab Blog](http://blog.gitlab.org/) for installation guide links by version.

# Important notes

This installation guide was created for and tested on **Debian/Ubuntu** operating systems. Please read [`doc/install/requirements.md`](./requirements.md) for hardware and operating system requirements.

This is the official installation guide to set up a production server. To set up a **development installation** or for many other installation options please consult [the installation section in the readme](https://github.com/gitlabhq/gitlabhq#installation).

The following steps have been known to work. Please **use caution when you deviate** from this guide. Make sure you don't violate any assumptions GitLab makes about its environment. For example many people run into permission problems because they changed the location of directories or run services as the wrong user.

If you find a bug/error in this guide please **submit a pull request** following the [contributing guide](../../CONTRIBUTING.md).

- - -

# Overview

The GitLab installation consists of setting up the following components:

1. Packages / Dependencies
2. Ruby
3. System Users
4. GitLab shell
5. Database
6. GitLab
7. Nginx


# 1. Packages / Dependencies

`sudo` is not installed on Debian by default. Make sure your system is
up-to-date and install it.

    # run as root!
    apt-get update -y
    apt-get upgrade -y
    apt-get install sudo -y

**Note:**
Vim is an editor that is used here whenever there are files that need to be
edited by hand. But, you can use any editor you like instead.

    # Install vim
    sudo apt-get install -y vim

Install the required packages:

    sudo apt-get install -y build-essential zlib1g-dev libyaml-dev libssl-dev libgdbm-dev libreadline-dev libncurses5-dev libffi-dev curl git-core openssh-server redis-server checkinstall libxml2-dev libxslt-dev libcurl4-openssl-dev libicu-dev

Make sure you have the right version of Python installed.

    # Install Python
    sudo apt-get install -y python

    # Make sure that Python is 2.5+ (3.x is not supported at the moment)
    python --version

    # If it's Python 3 you might need to install Python 2 separately
    sudo apt-get install python2.7

    # Make sure you can access Python via python2
    python2 --version

    # If you get a "command not found" error create a link to the python binary
    sudo ln -s /usr/bin/python /usr/bin/python2

**Note:** In order to receive mail notifications, make sure to install a
mail server. By default, Debian is shipped with exim4 whereas Ubuntu
does not ship with one. The recommended mail server is postfix and you can install it with:

	sudo apt-get install -y postfix 

Then select 'Internet Site' and press enter to confirm the hostname.

# 2. Ruby

Remove the old Ruby 1.8 if present

    sudo apt-get remove -y ruby1.8

Download Ruby and compile it:

    mkdir /tmp/ruby && cd /tmp/ruby
    curl --progress http://ftp.ruby-lang.org/pub/ruby/1.9/ruby-1.9.3-p392.tar.gz | tar xz
    cd ruby-1.9.3-p392
    ./configure
    make
    sudo make install

Install the Bundler Gem:

    sudo gem install bundler --no-ri --no-rdoc


# 3. System Users

Create a `git` user for Gitlab:

    sudo adduser --disabled-login --gecos 'GitLab' git


# 4. GitLab shell

GitLab Shell is a ssh access and repository management software developed specially for GitLab.

    # Go to home directory
    cd /home/git

    # Clone gitlab shell
    sudo -u git -H git clone https://github.com/gitlabhq/gitlab-shell.git

    cd gitlab-shell

    # switch to right version
    sudo -u git -H git checkout v1.4.0

    sudo -u git -H cp config.yml.example config.yml

    # Edit config and replace gitlab_url
    # with something like 'http://domain.com/'
    sudo -u git -H vim config.yml

    # Do setup
    sudo -u git -H ./bin/install


# 5. Database

To setup the MySQL/PostgreSQL database and dependencies please see [`doc/install/databases.md`](./databases.md).


# 6. GitLab

    # We'll install GitLab into home directory of the user "git"
    cd /home/git

## Clone the Source

    # Clone GitLab repository
    sudo -u git -H git clone https://github.com/gitlabhq/gitlabhq.git gitlab

    # Go to gitlab dir
    cd /home/git/gitlab

    # Checkout to stable release
    sudo -u git -H git checkout 5-3-stable

**Note:**
You can change `5-3-stable` to `master` if you want the *bleeding edge* version, but do so with caution!

## Configure it

    cd /home/git/gitlab

    # Copy the example GitLab config
    sudo -u git -H cp config/gitlab.yml.example config/gitlab.yml

    # Make sure to change "localhost" to the fully-qualified domain name of your
    # host serving GitLab where necessary
    sudo -u git -H vim config/gitlab.yml

    # Make sure GitLab can write to the log/ and tmp/ directories
    sudo chown -R git log/
    sudo chown -R git tmp/
    sudo chmod -R u+rwX  log/
    sudo chmod -R u+rwX  tmp/

    # Create directory for satellites
    sudo -u git -H mkdir /home/git/gitlab-satellites

    # Create directories for sockets/pids and make sure GitLab can write to them
    sudo -u git -H mkdir tmp/pids/
    sudo -u git -H mkdir tmp/sockets/
    sudo chmod -R u+rwX  tmp/pids/
    sudo chmod -R u+rwX  tmp/sockets/

    # Create public/uploads directory otherwise backup will fail
    sudo -u git -H mkdir public/uploads
    sudo chmod -R u+rwX  public/uploads

    # Copy the example Puma config
    sudo -u git -H cp config/puma.rb.example config/puma.rb

    # Enable cluster mode if you expect to have a high load instance
    # Ex. change amount of workers to 3 for 2GB RAM server
    sudo -u git -H vim config/puma.rb

    # Configure Git global settings for git user, useful when editing via web
    # Edit user.email according to what is set in gitlab.yml
    sudo -u git -H git config --global user.name "GitLab"
    sudo -u git -H git config --global user.email "gitlab@localhost"

**Important Note:**
Make sure to edit both `gitlab.yml` and `puma.rb` to match your setup.

## Configure GitLab DB settings

    # Mysql
    sudo -u git cp config/database.yml.mysql config/database.yml

    or

    # PostgreSQL
    sudo -u git cp config/database.yml.postgresql config/database.yml

    # Make sure to update username/password in config/database.yml.
    # You only need to adapt the production settings (first part).
    # If you followed the database guide then please do as follows:
    # Change 'root' to 'gitlab'
    # Change 'secure password' with the value you have given to $password
    # You can keep the double quotes around the password
    sudo -u git -H vim config/database.yml
    
    # Make config/database.yml readable to git only
    sudo -u git -H chmod o-rwx config/database.yml

## Install Gems

    cd /home/git/gitlab

    sudo gem install charlock_holmes --version '0.6.9.4'

    # For MySQL (note, the option says "without ... postgres")
    sudo -u git -H bundle install --deployment --without development test postgres

    # Or for PostgreSQL (note, the option says "without ... mysql")
    sudo -u git -H bundle install --deployment --without development test mysql


## Initialize Database and Activate Advanced Features

    sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production

    # Type 'yes' to create the database.

    # When done you see 'Administrator account created:'


## Install Init Script

Download the init script (will be /etc/init.d/gitlab):

    sudo cp lib/support/init.d/gitlab /etc/init.d/gitlab
    sudo chmod +x /etc/init.d/gitlab

Make GitLab start on boot:

    sudo update-rc.d gitlab defaults 21


## Check Application Status

Check if GitLab and its environment are configured correctly:

    sudo -u git -H bundle exec rake gitlab:env:info RAILS_ENV=production

## Start Your GitLab Instance

    sudo service gitlab start
    # or
    sudo /etc/init.d/gitlab restart

## Double-check Application Status

To make sure you didn't miss anything run a more thorough check with:

    sudo -u git -H bundle exec rake gitlab:check RAILS_ENV=production

If all items are green, then congratulations on successfully installing GitLab!
However there are still a few steps left.


# 7. Nginx

**Note:**
If you can't or don't want to use Nginx as your web server, have a look at the
[`Advanced Setup Tips`](./installation.md#advanced-setup-tips) section.

## Installation
    sudo apt-get install -y nginx

## Site Configuration

Download an example site config:

    sudo cp lib/support/nginx/gitlab /etc/nginx/sites-available/gitlab
    sudo ln -s /etc/nginx/sites-available/gitlab /etc/nginx/sites-enabled/gitlab

Make sure to edit the config file to match your setup:

    # Change YOUR_SERVER_FQDN to the fully-qualified
    # domain name of your host serving GitLab.
    sudo vim /etc/nginx/sites-available/gitlab

## Restart

    sudo service nginx restart


# Done!

Visit YOUR_SERVER for your first GitLab login.
The setup has created an admin account for you. You can use it to log in:

    admin@local.host
    5iveL!fe

**Important Note:**
Please go over to your profile page and immediately change the password, so
nobody can access your GitLab by using this login information later on.

**Enjoy!**


- - -


# Advanced Setup Tips

## Custom Redis Connection

If you'd like Resque to connect to a Redis server on a non-standard port or on
a different host, you can configure its connection string via the
`config/resque.yml` file.

    # example
    production: redis://redis.example.tld:6379

## Custom SSH Connection

If you are running SSH on a non-standard port, you must change the gitlab user's SSH config.

    # Add to /home/git/.ssh/config
    host localhost          # Give your setup a name (here: override localhost)
        user git            # Your remote git user
        port 2222           # Your port number
        hostname 127.0.0.1; # Your server name or IP

You also need to change the corresponding options (e.g. ssh_user, ssh_host, admin_uri) in the `config\gitlab.yml` file.

## LDAP authentication

You can configure LDAP authentication in config/gitlab.yml. Please restart GitLab after editing this file.

## Using Custom Omniauth Providers

GitLab uses [Omniauth](http://www.omniauth.org/) for authentication and already ships with a few providers preinstalled (e.g. LDAP, GitHub, Twitter). But sometimes that is not enough and you need to integrate with other authentication solutions. For these cases you can use the Omniauth provider.

### Steps

These steps are fairly general and you will need to figure out the exact details from the Omniauth provider's documentation.

* Add `gem "omniauth-your-auth-provider"` to the [Gemfile](https://github.com/gitlabhq/gitlabhq/blob/5-3-stable/Gemfile#L18)
* Run `sudo -u git -H bundle install` to install the new gem(s)
* Add provider specific configuration options to your `config/gitlab.yml` (you can use the [auth providers section of the example config](https://github.com/gitlabhq/gitlabhq/blob/5-3-stable/config/gitlab.yml.example#L53) as a reference)
* Add icons for the new provider into the [vendor/assets/images/authbuttons](https://github.com/gitlabhq/gitlabhq/tree/5-3-stable/vendor/assets/images/authbuttons) directory (you can find some more popular ones over at https://github.com/intridea/authbuttons)
* Restart GitLab

### Examples

If you have successfully set up a provider that is not shipped with GitLab itself, please let us know.
You can help others by reporting successful configurations and probably share a few insights or provide warnings for common errors or pitfalls by sharing your experience [in the public Wiki](https://github.com/gitlabhq/gitlab-public-wiki/wiki/Working-Custom-Omniauth-Provider-Configurations).
While we can't officially support every possible auth mechanism out there, we'd like to at least help those with special needs.
