+++
date = '2025-01-10T15:19:01-07:00'
draft = true
title = 'Gems'
+++
Like many Ruby developers focused on Rails, I have used Gems as long as I have been writing Ruby, but it dawned on my that I don't really have an idea of what Gems are, so this article is to explore that. 
## What is a Gem and how are they written? 
Gems are how we package code for reuse in other projects in the Ruby ecosystem. 
The directory structure for a Gem called SomeGem could look like this:
```
bin/
	... the executables
lib/
	some_gem/
		... other files
	some_gem.rb
test/
	... the tests
some_gem.gemspec
```

The code for the Gem is housed in `lib`, the executables under `bin`, the tests in `test`, and the specifications (name, version, dependencies, etc) in the `gemspec` file.

If you follow a basic tutorial on creating a gem, like RubyGems.org provides, you will find that building the gem creates a file called something like `SomeGem-0.0.0.gem`. This is an archive of the source code and metadata pertaining to that version of the gem. Extracting it shows a few files that can be explored

```shell
$ some_gem file SomeGem-0.0.0.gem
SomeGem-0.0.0.gem: POSIX tar archive
$ some_gem tar -xvf SomeGem-0.0.0.gem
x metadata.gz
x data.tar.gz
x checksums.yaml.gz
$ some_gem gunzip metadata.gz
$ some_gem cat metadata
# the stuff from the gemspec
$ some_gem gunzip data.tar.gz && tar -xvf data.tar
# data.tar is the archive of the source code of the gem
```
## How do you use a gem?
After you have installed a gem to a properly configured Ruby environment (more on that later) you interact with it by requiring it in your code. `require` is a method built in to Ruby but it is modified by the presence of `RubyGems`, which has been packaged with Ruby for a long time. `require`ing a gem means that gem's `lib` directory is added to your `$LOAD_PATH`. The `$LOAD_PATH` is an array of strings representing directories in the local environment. 
The following will add the `lib/` directory of SomeGem to the `$LOAD_PATH`, making the code and classes there available. 
```
$ irb
>> require 'some_gem'
=> true
>> puts $LOAD_PATH
=> 
[
...
your-ruby-install/gems/SomeGem-0.0.1/lib
]
>> SomeGem.something
=> something
```

## Where does Bundler come into this? 
Most of us are used to interacting with gems through the Gemfile. You can see in the above code that my example gem was installed to the directory where the current version of Ruby installed, but in real projects we want to scope the gem to the current project. Bundler provides us with a place to keep track of the gems our project uses, where to find them, and their versions. Running `bundle init` in a directory without a Gemfile adds one, so we can specify our own gems to use in this project:
```shell
$ mkdir bundler_example
$ cd bundler_example
$ bundler_example bundle init
Writing new Gemfile to /Users/jakebillsdev/tutorials/bundler_example/Gemfile
$ bundler_example tree
.
└── Gemfile

1 directory, 1 file
$ bundler_example cat Gemfile
# frozen_string_literal: true

source "https://rubygems.org"

# gem "rails"
$ bundler_example echo 'gem "some_gem"' >> Gemfile
$ bundler_example cat Gemfile
# frozen_string_literal: true

source "https://rubygems.org"

# gem "rails"
gem "some_gem"
```


## Sources
https://stackoverflow.com/questions/15308970/what-does-a-gem-file-contain-how-it-is-being-used-by-rails-framework
https://thoughtbot.com/blog/following-the-path
https://guides.rubygems.org/what-is-a-gem/
https://bundler.io/guides/getting_started.html