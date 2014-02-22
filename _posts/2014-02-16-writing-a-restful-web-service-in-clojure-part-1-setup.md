---
layout: post
title: "Writing a RESTful web service in Clojure | Part 1: Setup"
description: "Create a RESTful web service project in Clojure using Compojure, Liberator, and Korma"
category: restful-clojure
tags: ["clojure", "REST", "Puppet", "Leiningen", "tutorial"]
---
{% include JB/setup %}

Over the next several posts, we'll be doing and end-to-end walkthrough of
creating a RESTful web service using the [Clojure](http://clojure.org/)
programming language. Unlike a lot of webapp tutorials on the web, this
tutorial series will focus on the entire project as a whole, including
infrastructure setup using [Vagrant](http://www.vagrantup.com/) and
[Puppet](http://puppetlabs.com/puppet/puppet-open-source), testing/CI, and
deploying to [Heroku](https://www.heroku.com/).

The project that we will be building is a shopping list application. The basic
operation will be fairly simple, but it will allow us to tackle most of the
issues that we'll run into building RESTful web services professionally. At the
end of the tutorial series, we'll build a simple javascript client application
so that we can actually enjoy the fruits of our labour. You should be able to
extend the application to create an awesome web or mobile app.

All of the source code for this project is available as a 
[GitHub repo](https://github.com/kendru/restful-clojure). Each part of the
project has a corresponding branch that you can checkout for that specific
part. For example, for this first tutorial, you can checkout `part2`.
Please fork the project and experiment with it. I'm releasing all of the code
under a MIT license, so you can do pretty much anything you want with it.

## Project setup

In this first tutorial, we'll be setting up a development environment with
Vagrant and Puppet, and we will set up our initial project with
[Leiningen](http://leiningen.org/). I have seen a number of projects that
started out small then grew too big too fast, retaining an ad-hoc build process
and a development environment that took days to get set up. For our project,
we'll avoid that technical debt by manging our development environments with
Vagrant (using Puppet for provisioning) and managing our build with the
excellent Leiningen build tool for Clojure.

Let's start out by downloading Leiningen and creating a new project. Leiningen
is a build tool written in Clojure and designed specifically for Clojure
projects. Over the course of this tutorial, we'll be using it for running
tests, controlling database migrations, and packaging our application for
deployment. Its plugin system makes it suitable to be a general-purpose task
runner, so any process that you find yourself repeating often while developing
an app can be easily automated with Leiningen. If you have used ruby's Rake or
a similar tool, you should be able to pick up on Leiningen very quickly.
Although we won't go into Leiningen internals in this tutorial series, I would
strongly encourage you to read over the documentation and even dig into the
code - it is an awesome tool that I have found invaluable for developing in
Clojure. With that brief introduction, let's dig in!

You'll need to download the lein script and place it somewhere on your path.
Assuming you have a `bin` subdirectory inside your home directory, the
following commands are all that is necessary to get started:

{% highlight bash %}
wget https://raw.github.com/technomancy/leiningen/stable/bin/lein -O ~/bin/lein
chmod a+x ~/bin/lein
{% endhighlight %}

With that, we can create our project with one command:
{% highlight bash %}
lein new restful-clojure
{% endhighlight %}

This will create the basic project structure that we will be building our app
on. I have removed the LICENSE and README.md files that Leiningen generates and
have placed a license and readme in the project root instead. Lein works under
the assumption that the project forder that it generates will be the root of
a project, but since we're keeping Vagrant/Puppet configuration in our project
as well, our project root will actually be the parent folder to the folder that
Leiningen generates. Right now, your project should look something like the
following:

```
├── LICENSE
├── README.md
└── restful-clojure
    ├── doc
    ├── project.clj
    ├── resources
    ├── src
    └── test
```

Next up, let's install Vagrant. For this tutorial, I am running 1.4.3,
but any 1.4 release should work. Head over to the 
[Vagrant downloads](http://www.vagrantup.com/downloads) page to find the
package for your OS. Vagrant uses
[VirtualBox](https://www.virtualbox.org/wiki/Downloads) to host its VMs, so please
download the latest version. Once you have Vagrant and VirtualBox installed,
grab the project source from GitHub:

{% highlight bash %}
git clone https://github.com/kendru/restful-clojure
cd restful-clojure
git checkout tags/part1
{% endhighlight %}

At this point, you can run `vagrant up` to download the vagrant box and boot
your dev VM. Since the box is about 400MB, go ahead and start Vagrant so that
the download will start. In the meantime, let's take a look at the Vagrantfile:

{% highlight ruby %}
config.vm.box = "ubuntu-puppetlabs"
config.vm.box_url = "http://puppet-vagrant-boxes.puppetlabs.com/ubuntu-server-12042-x64-vbox4210.box"
{% endhighlight %}

Here we are instructing Vagrant to download the box from the Puppetlabs URL as
"ubuntu-puppetlabs". If you want to use another box, you can easily specify
the box name and URL here.

The next settings we'll tweak are the network settings. We'll assign the VM an
IP on a private network so that we can treat it almost like a remote machine
and test our deployment using the VM. We also allow the machine to access our
host machine's network so that we can take advantage of our host's internet
connection for downloading the packages that we want to install.

{% highlight ruby %}
config.vm.network :private_network, ip: "192.168.33.10"
config.vm.network :public_network
{% endhighlight %}

We also want to be able to access our project from within the VM, so we'll want
to mount our Leiningen project folder within the VM.
{% highlight ruby %}
config.vm.synced_folder "./restful-clojure", "/vagrant"
{% endhighlight %}

Finally, we configure Vagrant to provision our VM using Puppet. We create
a `puppet` folder with `manifests` and `modules` subfolders.

{% highlight bash %}
mkdir puppet puppet/manifests puppet/modules
{% endhighlight %}

For our app, our only dependencies (at least initially) will be a Java runtime
and the PostgreSQL database. Instead of manually writing all of the boilerplate
Puppet code for setting up Java and Postgres, we'll use Puppet modules, which
we'll add to our repo as git submodules If you have cloned the project from
github, you can get the submodules by executing:
{% highlight bash %}
git submodule update --init --recursive
{% endhighlight %}
Otherwise, add all of the puppet modules that we need as git submodules within
puppet/modules:
{% highlight bash %}
git submodule add https://github.com/puppetlabs/puppetlabs-stdlib.git puppet/modules/stdlib
git submodule add https://github.com/puppetlabs/puppetlabs-postgresql.git puppet/modules/postgresql
git submodule add https://github.com/puppetlabs/puppetlabs-concat.git puppet/modules/concat
git submodule add https://github.com/softek/puppet-java7 puppet/modules/java7
git submodule add https://github.com/puppetlabs/puppetlabs-apt.git puppet/modules/apt
{% endhighlight %}

Finally, we'll create a Puppet manifest at
puppet/manifests/default.pp and configure Vagrant to use this manifest for
provisioning the VM:

#### Vagrantfile
{% highlight ruby %}
config.vm.provision "puppet" do |puppet|
	puppet.manifests_path = "puppet/manifests"
	puppet.module_path = "puppet/modules"
	puppet.manifest_file = "default.pp"
end
{% endhighlight %}

#### puppet/manifests/default.pp
{% highlight ruby %}
exec { 'update_packages':
	command => 'apt-get update',
	path    => '/usr/bin',
}

# Install Leiningen to run tests and build project
exec { 'get_leiningen':
	command => '/usr/bin/wget https://raw.github.com/technomancy/leiningen/stable/bin/lein -O /usr/bin/lein && /bin/chmod a+x /usr/bin/lein',
	unless  => '/usr/bin/which lein &>/dev/null',
}


class { 'postgresql::server': }

postgresql::server::db { 'restful_dev':
	user     => 'restful_dev',
	password => postgresql_password('restful_dev', 'pass_dev'),
}

postgresql::server::db { 'restful_test':
	user     => 'restful_test',
	password => postgresql_password('restful_test', 'pass_test'),
}

include java7
{% endhighlight %}

From this point, fire up your VM, and you should have your environment
available:
{% highlight bash %}
# If you have not yet started your vagrant box
vagrant up
# If you have already run 'vagrant up'
vagrant provision
{% endhighlight %}

Congratulations - you now have a functional Leiningen project set up with
a shiny new development VM to run it on! In [the next tutorial](/restful-clojure/2014/02/19/getting-a-web-server-up-and-running-with-compojure-restful-clojure-part-2/), we'll take our
application from skeleton to basic web server with a few tests to boot.
