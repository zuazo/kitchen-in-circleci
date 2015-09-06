kitchen-in-circleci Cookbook [![Circle CI](https://circleci.com/gh/zuazo/kitchen-in-circleci/tree/master.svg?style=shield)](https://circleci.com/gh/zuazo/kitchen-in-circleci/tree/master)
============================

Proof of concept cookbook to run [test-kitchen](http://kitchen.ci/) inside [CircleCI](https://circleci.com/) using [kitchen-docker](https://github.com/portertech/kitchen-docker) gem.

You can use this in your cookbook by using a *circle.yml* file similar to the following:

```yaml
machine:
  services:
  - docker
  ruby:
    version: 2.2.2

dependencies:
  override:
  - bundle check --path=vendor/bundle || bundle install --path=vendor/bundle --jobs=4 --retry=3:
      timeout: 900

test:
  override:
  - KITCHEN_LOCAL_YAML=.kitchen.docker.yml bundle exec kitchen test:
      timeout: 900
```

Look [below](https://github.com/zuazo/kitchen-in-circleci#how-to-implement-this-in-my-cookbook) for more complete examples.

The following files will help you understand how this works:

* [*circle.yml*](https://github.com/zuazo/kitchen-in-circleci/blob/master/circle.yml)
* [*.kitchen.docker.yml*](https://github.com/zuazo/kitchen-in-circleci/blob/master/.kitchen.docker.yml)
* [*Rakefile*](https://github.com/zuazo/kitchen-in-circleci/blob/master/Rakefile)

This example cookbook only installs nginx. It also includes some [Serverspec](http://serverspec.org/) tests to check everything is working correctly.

## Install the Requirements

First you need to install [Docker](https://docs.docker.com/installation/).

Then you can use [bundler](http://bundler.io/) to install the required ruby gems:

    $ gem install bundle
    $ bundle install

## Running the Tests in Your Workstation

    $ bundle exec rake

This example will run kitchen **with Vagrant** in your workstation. You can use `$ bundle exec rake integration:docker` to run kitchen with Docker, as in CircleCI.

## Available Rake Tasks

    $ bundle exec rake -T
    rake integration:docker[regexp,action]   # Run integration tests with kitchen-docker
    rake integration:vagrant[regexp,action]  # Run integration tests with kitchen-vagrant

## How to Implement This in My Cookbook

First, create a `.kitchen.docker.yml` file with the platforms you want to test:

```yaml
---
driver:
  name: docker

platforms:
- name: centos-6.6
  run_list:
- name: ubuntu-14.04
  run_list:
  - recipe[apt]
# [...]
```

If not defined, it will get the platforms from the main `.kitchen.yml` by default.

You can get the list of the platforms officially supported by Docker [here](https://registry.hub.docker.com/repos/library).

Then, I recommend you to create a task in your *Rakefile*:

```ruby
# Rakefile
require 'bundler/setup'

# [...]

desc 'Run Test Kitchen integration tests'
namespace :integration do
  desc 'Run integration tests with kitchen-docker'
  task :docker do
    require 'kitchen'
    Kitchen.logger = Kitchen.default_file_logger
    @loader = Kitchen::Loader::YAML.new(local_config: '.kitchen.docker.yml')
    Kitchen::Config.new(loader: @loader).instances.each do |instance|
      instance.test(:always)
    end
  end
end
```

This will allow us to use `$ bundle exec rake integration:docker` to run all the tests.

The *circle.yml* file example:

```yaml
machine:
  services:
  - docker
  ruby:
    version: 2.2.2

dependencies:
  override:
  - bundle check --path=vendor/bundle || bundle install --path=vendor/bundle --jobs=4 --retry=3:
      timeout: 900

test:
  override:
  - bundle exec rake integration:docker
      timeout: 900
```

If you are using a *Gemfile*, you can add the following to it:

```ruby
# Gemfile

group :integration do
  gem 'test-kitchen', '~> 1.2'
end

group :docker do
  gem 'kitchen-docker', '~> 2.1.0'
end
```

This will be enough if you want to test only 2 or 3 platforms. If you want more, continue reading:

### How to Run Tests in Many Platforms

If you want to test many platforms, you will need to split up the tests in multiple CircleCI builds. For those cases, I recommend you to use a *Rakefile* Rake task similar to the following:

```ruby
# Rakefile
require 'bundler/setup'

# [...]

desc 'Run Test Kitchen integration tests'
namespace :integration do
  # Generates the `Kitchen::Config` class configuration values.
  #
  # @param loader_config [Hash] loader configuration options.
  # @return [Hash] configuration values for the `Kitchen::Config` class.
  def kitchen_config(loader_config = {})
    {}.tap do |config|
      unless loader_config.empty?
        @loader = Kitchen::Loader::YAML.new(loader_config)
        config[:loader] = @loader
      end
    end
  end

  # Gets a collection of instances.
  #
  # @param regexp [String] regular expression to match against instance names.
  # @param config [Hash] configuration values for the `Kitchen::Config` class.
  # @return [Collection<Instance>] all instances.
  def kitchen_instances(regexp, config)
    instances = Kitchen::Config.new(config).instances
    return instances if regexp.nil? || regexp == 'all'
    instances.get_all(Regexp.new(regexp))
  end

  # Runs a test kitchen action against some instances.
  #
  # @param action [String] kitchen action to run (defaults to `'test'`).
  # @param regexp [String] regular expression to match against instance names.
  # @param loader_config [Hash] loader configuration options.
  # @return void
  def run_kitchen(action, regexp, loader_config = {})
    action = 'test' if action.nil?
    require 'kitchen'
    Kitchen.logger = Kitchen.default_file_logger
    config = kitchen_config(loader_config)
    kitchen_instances(regexp, config).each { |i| i.send(action) }
  end

  desc 'Run integration tests with kitchen-vagrant'
  task :vagrant, [:regexp, :action] do |_t, args|
    run_kitchen(args.action, args.regexp)
  end

  desc 'Run integration tests with kitchen-docker'
  task :docker, [:regexp, :action] do |_t, args|
    run_kitchen(args.action, args.regexp, local_config: '.kitchen.docker.yml')
  end
end
```

This will allow us to run different kitchen tests using the `$ rake integration:docker[REGEXP]` command. For example we can use `$ bundle exec rake integration:docker[ubuntu]` to run only the Ubuntu integration tests.

Then, you can use the following *circle.yml* file:

```yaml
machine:
  services:
  - docker
  ruby:
    version: 2.2.2
  environment:
    TESTS: ubuntu centos

dependencies:
  override:
  - bundle check --path=vendor/bundle || bundle install --path=vendor/bundle --jobs=4 --retry=3:
      timeout: 900

test:
  override:
  - TESTS=(${TESTS// / }) ; bundle exec rake integration:docker[${TESTS[$CIRCLE_NODE_INDEX]}]:
      parallel: true
      timeout: 900
```

This will allow us to configure the integration tests to run in parallel in the `TESTS` environment variable.

For this to work, you need to go to **Project Settings -> Tweaks -> Adjust Parallelism** in the CircleCI dashboard and set the paralellism to the number of tests (`2` in our case: `ubuntu` and `centos`).

## Known Issues

### Official CentOS 7 and Fedora Images

Cookbooks requiring [systemd](http://www.freedesktop.org/wiki/Software/systemd/) may not work correctly on CentOS 7 and Fedora containers. See [*Systemd removed in CentOS 7*](https://github.com/docker-library/docs/tree/master/centos#systemd-integration).

You can use alternative images that include systemd. These containers must run in **privileged** mode:

```yaml
# .kitchen.docker.yml

# Non-official images with systemd
- name: centos-7
  driver_config:
    # https://registry.hub.docker.com/u/milcom/centos7-systemd/dockerfile/
    image: milcom/centos7-systemd
    privileged: true
- name: fedora
  driver_config:
    image: fedora/systemd-systemd
    privileged: true
```

### Problems with Upstart in Ubuntu

Some cookbooks requiring [Ubuntu Upstart](http://upstart.ubuntu.com/) may not work correctly.

You can use the official Ubuntu images with Upstart enabled:

```yaml
# .kichen.docker.yml

- name: ubuntu-14.10
  run_list: recipe[apt]
  driver_config:
    image: ubuntu-upstart:14.10
```

### Install `netstat` Package

It's recommended to install `net-tools` on some containers if you want to test listening ports with Serverspec. This is because some images come without `netstat` installed.

This is required for example for the following Serverspec test:

```ruby
# test/integration/default/serverspec/default_spec.rb
describe port(80) do
  it { should be_listening }
end
```

You can ensure that `netstat` is properly installed running the [`netstat`](https://supermarket.chef.io/cookbooks/netstat) cookbook:

 ```yaml
# .kitchen.docker.yml

- name: debian-6
  run_list:
  - recipe[apt]
  - recipe[netstat]
```

### CircleCI Error: *We got an error (-1 - Request timed out.)*

If a command can take a long time to run and is very quiet, you may need to run it with some flags to increase verbosity such as: `--verbose`, `--debug`, `--l debug`, ... You can also increase the `timeout:` value in the *circle.yml* file.

## Feedback Is Welcome

Currently I'm using this for my own projects. It may not work correctly in many cases. If you use this or a similar approach successfully with other cookbooks, please [open an issue and let me know about your experience](https://github.com/zuazo/kitchen-in-circleci/issues/new). Problems, discussions and ideas for improvement, of course, are also welcome.

## Acknowledgements

This cookbook example does not contain anything new. It is based on [Torben Knerr](https://github.com/tknerr)'s work on the [`sample-toplevel-cookbook`](https://github.com/tknerr/sample-toplevel-cookbook) cookbook.

# License and Author

|                      |                                          |
|:---------------------|:-----------------------------------------|
| **Author:**          | [Xabier de Zuazo](https://github.com/zuazo) (<xabier@zuazo.org>)
| **Copyright:**       | Copyright (c) 2015, Xabier de Zuazo
| **License:**         | Apache License, Version 2.0

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at
    
        http://www.apache.org/licenses/LICENSE-2.0
    
    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
