# Writing your first cookbook

## Jackchop!
Alright you want to create your first cookbook. You followed all of the instructions [here](README.md) and have all the necessary tools at your disposal. It's a good idea to have a central place to keep all of your cookbooks, the Fellowship follows the convention of placing them in `~/projects/cookbooks` and we highly recommend you do the same.

Once you are in that directory run the following command -

```bash
jackchop cookbook create <name of your cookbook>
```

This should create a new directory in `cookbooks` named whatever you passed in to jackchop. Take a look in that directory and we'll see what was created.

You should see the following directories -
* `attributes` - This is where you still store the default attributes for your cookbook
* `recipes` - Recipes are the 'logic' of your cookbook
* `tests` - Should be pretty obvious what goes in this directory =)

The following files are also created -
* `.kitchen.yml` - Used to configure the test-kitchen framework used for integration testing
* `Berksfile` - This is the file used by Berkshelf to determine dependencies of your cookbook during development/testing.
* `chefignore` - This file tells the chef server what files to ignore during uploads and syntax checking.
* `metadata.rb` - Used to describe the cookbook

Next, issue the following commands and make sure everything passes -

```bash
bundle install
```

```bash
rake test
```

**NOTE:** If you have not installed vagrant yet, running `rake test` will fail. You can download the latest version of vagrant from [here](http://downloads.vagrantup.com/). You also need the vagrant-berkshelf plugin, to install run `vagrant plugin install vagrant-berkshelf`.

Jackchop will also init a git repo for you when you create a cookbook. **However, it does not create a github repo!** Currently, all cookbooks are under the RallySoftware-cookbooks organization which means the repo will need to be created by a member of the Fellowship team. This functionality may eventually be extended to Jarvis, but for now we will need to help you.

**Your cookbook is now the baddest in all of Revere!**

## Berksfile and metadata.rb
Before we talk about writing your first recipe I want to describe the purpose of the Berksfile and the metadata.rb file.
### Berksfile
The Berksfile is used by [Berkshelf](http://berkshelf.com/) to help manage the dependencies of your cookbook during development and test. It will allow you to specify a cookbook dependency as a git repo, a path on your local machine, or as version (using the bundler syntax).

```ruby
chef_api :config
site :opscode

metadata

cookbook 'memcached', '~> 0.1.0'
cookbook 'foo', path: '/Users/localadmin/projects/cookbooks/foo'
cookbook 'mysql', github: 'opscode-cookbooks/mysql'
cookbook 'mysql', git: 'git://github.com/opscode-cookbooks/mysql.git'
cookbook 'mysql', git: 'https://github.com/opscode-cookbooks/mysql.git', branch: 'foodcritic'
cookbook 'mysql', git: 'https://github.com/opscode-cookbooks/mysql.git', ref: 'eef7e65806e7ff3bdbe148e27c447ef4a8bc3881'
```

The first line (`chef_api :config`) in this mock `Berksfile` says to use the Berkshelf config (located at `~/.berkshelf/config.json`) to learn how to communicate with your Chef Server. In this particular case, it will end up using your knife config to reach out and fetch the cookbooks.

Next, `site :opscode` says to use the community site to fulfill any cookbooks that could not be found on your Chef Server and locations are not specified for the cookbook itself.

Finally, we see `metadata` which tells Berkshelf to use the `metadata.rb` file to locate additional dependencies that are not specified in the `Berksfile`.

### Metadata.rb
The `metadata.rb` file is what is used by Chef to resolve dependencies when the `chef-client` is executing on a node. This means that you MUST specify your dependent cookbooks in this file for them to be included when the node is converged.

```ruby
name             'foo'
maintainer       'Rally Software Development Corp'
maintainer_email 'rallysoftware-cookbooks@rallydev.com'
license          'MIT'
description      'Installs/Configures foo'
long_description IO.read(File.join(File.dirname(__FILE__), 'README.md'))
version          IO.read(File.join(File.dirname(__FILE__), 'VERSION'))

supports 'centos'
supports 'ubuntu'

depends 'bar'
depends 'baz', '~> 0.2.0'
```

The first section will be setup correctly when you use `jackchop` to create your cookbook. However, you will need to explicitly set the version each time you want to bump the version of your cookbook.

The `metadata.rb` has a `supports` tag for describing supported operating systems. While this is not used to control if a cookbook will actually run on a given OS, it is good to set these values for operating systems that are being tested against.

Lastly, the `depends` operator is used to express cookbook dependencies that need to be resolved on the node at converge time. It is not as expressive as the `Berkshelf` syntax is, however, it does support the bundler syntax for resolving symvers.

## Writing recipes
Alright so now you know how dependencies are handled, you can start working on your first recipe.

### The default recipe
Any cookbook development should always start with the default recipe. When `jackchop` created your cookbook a default recipe was automatically created and placed in the recipes directory.

First, cookbooks are a collection of [resources](http://docs.opscode.com/resource.html). `Resources` act like a DSL for providing logic to your recipe. Chef provides many resources out of the box such as -

* including other cookbook recipes
* creating [users](http://docs.opscode.com/resource_user.html)
* creating [groups](http://docs.opscode.com/resource_group.html)
* downloading files from [remote locations](http://docs.opscode.com/resource_remote_file.html)
* creating files from a ruby erb [templates](http://docs.opscode.com/resource_template.html)
* installing [packages](http://docs.opscode.com/resource_package.html)

You can also create your own resources, however, that is not covered in this tutorial.

#### Including other cookbook recipes within your own
In most cases you should only be worried about including other cookbook's recipes within your own. To do that you can use the `include_recipe` resource.

To include the default recipe you can just specify the cookbook.
```ruby
include_recipe 'mysql'
```

If you need to include a non-default recipe you use the `cookbook::recipe` syntax.
```ruby
include_recipe 'mysql::server'
```

At compile time, the chef-client will add the `server` recipe from the `mysql` cookbook into the runlist. This happens inline with the currently executing recipe. If you include a recipe in your cookbook you need to make sure that you define that dependency within your `metadata.rb`.

```ruby
depends 'mysql'
```

#### Using other [resources](http://docs.opscode.com/resource.html)
Creating a [user](http://docs.opscode.com/resource_user.html) is pretty common in all chef cookbooks. So we will use it as a guide for describing the general structure of Chef resources.

```ruby
user 'name'
  attribute  'value'
  attribute1 123
  action     :create
end
```

The first item after defining the resource is the `name` attribute. It is used as output during the chef run to mark which resource is being executed. For the user resource it will double as the username if the username attribute is not specified. This is common amongst all resources. You should be aware of what a resource will do with the name attribute when specified.

An Attribute for a resource can be any of the ruby primary types (strings, hashes, arrarys, booleans, integers) and those are specified in the attribute's resource file.

Lastly, the action attribute can be specified to invoke specific actions against a resource. For example, the create action on a user will create that user with the specified attributes. However, if you needed to remove a user you could specify the `:remove` attribute. Each resource *should* have a default action associated with it.

#### Examples of common resources
```ruby
package 'mysql' do
  action :install
end
```

```ruby
user 'username' do
  password 'Password'
end
```

```ruby
directory '/my/awesome/directory'
  action :create
  mode   '755'
  owner 'owner'
  group 'group'
end
```

```ruby
file '/where/to/place/my/file.foo'
  content 'my content'
  owner   'root'
  group   'root'
  mode    '755'
end
```

```ruby
remote_file '/path/to/place/file' do
  source 'http://foo.com/bar.zip'
  owner  'root'
  group  'root'
  mode   '755'
end
```

```ruby
template '/myservice/myservice.conf' do
  source 'myservice.conf.erb'
  mode '755'
  variables(:key => value, :key2 => value2)
end
```

## [Attributes](http://docs.opscode.com/essentials_cookbook_attribute_files.html)
Attributes represent a way to set configuration values for the software that is being installed on a given node. The values are set at converge time of the node and may be overriden through an order of precedence system. Attribute values may be unique to a given node or span all of the nodes in an orginization.

### `attributes/default.rb`
The most common way of setting attributes is through the `attributes/default.rb` file located in most cookbooks. Typically, this file will contain the sensible defaults for the cookbook and should allow for convergence on most systems. Values in this file may look like the following - 

```ruby
default[:myapp][:some_property][:another_property] = 'foo'
default[:myapp][:server][:should_install_bar] = true
```

**NOTE:** prefacing the attribute with default sets an order of precedence for the value. More on this later...

### `ohai` (pronounced oh hi)
[Ohai](http://docs.opscode.com/ohai.html) is a tool that is installed on all of the nodes that are bootstrapped with chef. It allows for cookbooks (and knife) to query information about the system that may not actually reside within a cookbook. For example you could retrieve -
* hostname
* fqdn (fully qualified domain name)
* ip address
* drive information
* file system mounts
* memory information
* what programming languages are installed 
* kernel information

The amount of information that can be retrieved from ohai is almost endless and it should be used any time you rely on system information for installing or upgrading software on a node.

### Roles and Environments
Roles and Environments represent 2 other ways that attributes can be set on a node. When attributes are set at this level it typically means they are more global in nature and may affect multiple nodes. If you feel that you have an attribute(s) that should be set at the role or environment level please come by and talk with the Fellowship team.

### Setting attributes in a recipe
You can also set attributes during the runtime of a recipe. This might happen if you need to compute some specific values based on the state of the system and use them later in the same recipe (or even in a different one).

Typically, you would want to set an attribute using the following syntax -

```ruby
node.default[:myattribute][:bar] = 'foo'
```

### Order of precedence
There is a complicated scheme to determine when values will be overriden. You can find a chart describing the order [here](http://docs.opscode.com/_images/overview_chef_attributes_table.png). I would also recommend reading the page on [attributes](http://docs.opscode.com/essentials_cookbook_attribute_files.html).

## Application cookbooks and attributes
Your cookbooks should follow the `application cookbook` pattern which means that you build 1 cookbook that encompasses everything that is needed for your application to run. In this cookbook you will set all of the default values that are required for your cookbook to converge and run the application.

For a detailed example please see the [buildserver cookbook](http://github.com/RallySoftware-cookbooks/buildserver).

## Writing tests
Ok, so you're done working on your cookbook (or, even better, maybe you haven't started yet) and you want to write some sweet tests. Good for you! Because cookbooks are pure ruby and will be executed in a production environment at some point, you will want to make sure they work correctly from the start. The best way to do that is to write tests that cover the logic that is being used within your cookbook.

We use 2 frameworks for testing cookbooks - `chefspec` and `test-kitchen`. When you used `jackchop` to create your cookbook it automatically setup these frameworks for you so there is nothing to do here on your part - except write tests!

### Unit tests - [chefspec](https://github.com/acrmp/chefspec)
Chefspec is an RSpec-based test framework which will exercise the logic in your cookbook without actually converging. It should be used to ensure that the conditional logic in your recipes are only being exercised in the correct circumstances. It is generally considered wasteful to write tests that simply [restate static resources](https://github.com/acrmp/chefspec#examples-should-do-more-than-restate-static-resources), rather you should test the conditions in which they are used, including their varying inputs.

For example, say that you write a config file from a template that will have different values when the cookbook executes on Ubuntu rather than CentOS. Testing the templates have these differences with fully converged VMs would be time-consuming at best, but being able to assert that the correct template resource and content will be created is sufficient to give us the confidence we need that the Chef run will be successful.

To run your unit tests execute the following command - `rake unit`.

For some good examples on writing `chefspec` tests look at the following -

* [Ad Auth](https://github.com/RallySoftware-cookbooks/ad-auth/blob/master/test/unit/default_spec.rb) tests show how to mock out a data bag and an environment
* [Ganglia](https://github.com/RallySoftware-cookbooks/ganglia/blob/master/test/unit/gmetad_spec.rb) shows how to test against templates
* [Selenium](https://github.com/RallySoftware-cookbooks/selenium/tree/master/test/unit) has a ton of tests around executing commands, working with cron, symlinks and much more

### Integration tests - [serverspec.org](http://serverspec.org/)
The bulk of our testing with cookbooks takes place using `serverspec`. Serverspec allows the user to write tests that verify the integrity of the operating system by executing commands and checking the standard out and return codes. These tests are facilitated by another framework called [Test Kitchen](https://github.com/opscode/test-kitchen). Test kitchen will stand up a vbox image and bootstrap that image with chef and make sure the cookbooks are uploaded to the in memory chef server (we use [chef-zero](https://github.com/jkeiser/chef-zero)).

#### `.kitchen.yml`
The kitchen yml file is the configurating file for your test kitchen environment. An example file make look like the following - 

```yml
---
driver_plugin: vagrant
provisoner: chef_zero
platforms:
- name: centos-6.4-x86_64
  driver_config:
    box: opscode-centos-6.4-x86_64
    box_url: https://opscode-vm.s3.amazonaws.com/vagrant/opscode_centos-6.4_chef-11.4.4.box
    require_chef_omnibus: 11.6.0
    customize:
      cpus: 1
      memory: 256
- name: ubuntu-12.04
  driver_config:
    box: opscode-ubuntu-12.04
    box_url: https://opscode-vm-bento.s3.amazonaws.com/vagrant/opscode_ubuntu-12.04_chef-11.4.4.box
    require_chef_omnibus: 11.6.0
    customize:
      cpus: 1
      memory: 256
suites:
- name: some_recipe_test
  run_list:
    - recipe[mycookbook::some_recipe]
    attributes:
    an_attribute:
      sub_attribute: 'my_value'
- name: another_recipe_test
  run_list:
    - recipe[mycookbook::another_recipe]
```

Test kitchen will create a permutation for each suite and platform provided in the kitchen yml. If you executed `kitchen list` in your cookbook directory it might look something like this -

```bash
some_recipe_test-centos-64-x86-64     Vagrant  Chef Zero    <Not Created>
some_recipe_test-ubuntu-1204          Vagrant  Chef Zero    <Not Created>
another_recipe_test-centos-64-x86-64  Vagrant  Chef Zero    <Not Created>
another_recipe_test-ubuntu-1204       Vagrant  Chef Zero    <Not Created>
```

When executing test kitchen commands in a mutli-suite/multi-platform environment you may want to specify a REGEX to make sure that only 1 suite of tests or 1 platform is executed against. For example -
* `kitchen converge some_recipe_test` - Run converge on the nodes that have `some_recipe_test` specified in the name
* `kitchen converge centos` - Just converge the `centos` nodes
* `kitchen verify some` - Converge and run the tests on nodes that have `some` in the name.

Get the idea?

#### Test kitchen commands
There are just a few commands that you should know -
* `kitchen converge` - Just converge the node (do not run the tests) - all by default
* `kitchen list` - List all the nodes that will be started - based on the permutation of platforms and suites
* `kitchen login` - Login to the node(s) specified by the regex
* `kitchen verify` - Converge and then run the tests but do not destroy the node
* `kitchen destroy` - Destroy the nodes specified by the regex (all by default)
* `kitchen test` - Run all the tests after the convergence - will destroy nodes when completed. If a test fails for a node it will hault execution. You can then use kitchen login for that node to inspect the state of the VM.

#### Writing Serverspec tests
For each suite in your kitchen yml there needs to be a directory that corresponds to that suite name.

* `test/integration/default/serverspec`
* `test/integration/my-other-suite/serverspec`

You can then have as many spec files as you want under the serverspec directory as you wish. 

Use the following cookbooks as examples when writing `serverspec` tests -
* [leiningen](https://github.com/RallySoftware-cookbooks/leiningen/blob/master/test/integration/leiningen/serverspec/leiningen_spec.rb)
* [ganglia](https://github.com/RallySoftware-cookbooks/ganglia/tree/master/test/integration/default/serverspec)
* [buildserver](https://github.com/RallySoftware-cookbooks/buildserver/tree/master/test/integration/default/serverspec)

## Writing a kick ass README!
Jackchop created a README.md file for you with some instructions on how to make it better. **PLEASE DO THIS!!**
**All cookbooks must have a README, it is required!**

## Publishing your cookbook to the chef server
Want to get your cookbook on to our Chef Server? Come talk with the fellowship and we will help you!
