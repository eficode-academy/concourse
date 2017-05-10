
**This is a modified version of the [Flight school](http://concourse.ci/flight-school.html) tutorial provided by Concourse.ci.**

**As with the original, this is relased under Apache-2.0.**

# Flight School


Welcome, Cadet.

This walkthrough assumes you've already got a Concourse set up and ready
to go. We have made a shared concourse available for this workshop. The URL will be shown on the slides.

We also assume that you have a GitHub account. If you don't yet, no worries. It takes only a minute to get one.

## Getting Set Up

First, let's get you a copy of your training materials.

Fork the repo [https://github.com/concourse/flight-school](https://github.com/concourse/flight-school) on
GitHub into your own account. This will give you somewhere to keep your
work.

Next, clone your fork down to your local machine by running this
command:



```sh
$ git clone https://github.com/(your username)/flight-school
$ cd flight-school
```


This is a simple [Ruby](https://www.ruby-lang.org) project that will let
you get a feel for flying. We don't actually need Ruby locally to work with this as Concourse will be doing all the heavy lifting.


## First Steps

Ok, we've got a project. We want a pipeline. It's important to build a
pipeline up gradually rather than attempting to do the whole thing at
once. Let's start at the start by running those unit tests we just ran
in Concourse.

Download `fly` from your Concourse. You can find a download link on your
Concourse installation main page. They will either be at the bottom
right or in the middle if you don't have any pipelines yet.


```sh
$ mkdir -p $HOME/bin
$ install $HOME/Downloads/fly $HOME/bin
```

Make sure that `$HOME/bin` is [in your
path](http://www.troubleshooters.com/linux/prepostpath.htm).

You should now be able to run `fly --help` to see if everything is
working.

Ok, let's target and log in to our Concourse.


```sh
$ fly -t meetup login -c (your concourse URL)
```

The `-t` flag is the name we'll use to refer to this instance in the
future. The `-c` flag is the concourse URL that we'd like to target.

Depending on the authentication setup of your Concourse it may prompt
you for various credentials to prove you are who you say you are.

Right, let's try running the current project in Concourse.


```sh
$ fly -t meetup execute
error: the required flag '-c, --config' was not specified
```


OK. Concourse wants to know what to execute. Well then, let's give it what it wants.


Let us create a file called `build.yml`.

We can't just give it an empty file. We need to write a task definition. A task definition describes a unit of
work to Concourse so that it can execute it.

Add the following content to the file:

```yaml
platform: linux

image_resource:
  type: docker-image
  source: 
    repository: busybox

inputs:
- name: flight-school

run:
  path: ./flight-school/ci/test.sh
```

Let's go through this line by line:

-   The [`platform`](http://concourse.ci/running-tasks.html#platform) simply states that we
    would like this task to run on a Linux worker.

-   The [`image_resource`](http://concourse.ci/running-tasks.html#image_resource) section
    declares the image to use for the task's container. It is defined as
    a [resource](http://concourse.ci/concepts.html#resources) configuration.

-   The [`inputs`](http://concourse.ci/running-tasks.html#inputs) section defines a set of
    things that we need in order for our task to run. In this case, we
    need the `flight-school` source code in order to run the tests on
    it.

-   The final [`run`](http://concourse.ci/running-tasks.html#task-run) section describes how
    Concourse should run the task. By default Concourse will run your
    script with a current working directory containing all of your
    inputs as subdirectories.

Ok, let's save that in `build.yml` and run our execute command again.


```sh
$ fly -t meetup execute -c build.yml
executing build 107401
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 40960    0 40960    0     0   662k      0 --:--:-- --:--:-- --:--:--  800k
initializing
running flight-school/ci/test.sh
exec failed: exec: "./flight-school/ci/test.sh": stat ./flight-school/ci/test.sh: no such file or directory
failed
```

Ok, so what happened here. We started a build, uploaded the
`flight-school` input, and then tried to run the
`flight-school/ci/test.sh` script. Which isn't there. Oops! Let's write
that.

```bash
mkdir ci
nano ci/test.sh
```

Add the following content:

```bash
#!/bin/bash

set -e -x

pushd flight-school
  bundle install
  bundle exec rspec
popd
```


This is basically the same commands we would use in order to run the
tests locally if we had Ruby installed. The new bits at the start set up a few things. The
`#!/bin/bash` is a [shebang
line](https://en.wikipedia.org/wiki/Shebang_(Unix)) that tells the
operating system that when we execute this file we should run it using
the `/bin/bash` interpreter. The `set -e -x` line is setting a few
`bash` options. Namely, `-e` make it so the entire script fails if a
single command fails (which is generally desirable in CI). By default, a
script will keep executing if something fails. The `-x` means that each
command should be printed as it's run (also desirable in CI).

Let's give this new script a whirl.



```sh
$ fly -t meetup execute -c build.yml
executing build 107401
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 40960    0 40960    0     0   662k      0 --:--:-- --:--:-- --:--:--  800k
initializing
running flight-school/ci/test.sh
exec failed: exec: "./flight-school/ci/test.sh": permission denied
failed
```



This error message means that the script we told it to run is not
executable. Let's fix that.



```sh
$ chmod +x ci/test.sh
```



Try running again. Notice the new error:



```sh
$ fly -t meetup execute -c build.yml
executing build 107401
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 40960    0 40960    0     0   662k      0 --:--:-- --:--:-- --:--:--  800k
initializing
running flight-school/ci/test.sh
exec failed: no such file or directory
failed
```


This message is a little obscure. It's complaining that the shebang
(`/bin/bash`) can't find the interpreter. In our task config, we
specified the `busybox` Docker image, which is a tiny, un-opinionated
operating system image that doesn't contain `bash`. This isn't very
useful for running builds so let's pick one that is.

Docker maintains a collection of Docker images for common languages.
Let's use the `ruby` image at version `2.3.0`. We can specify that the
task should run with this image by updating the
[`image_resource`](http://concourse.ci/running-tasks.html#image_resource) block in our
`build.yml` like so:



```yaml
image_resource:
  type: docker-image
  source:
    repository: ruby
    tag: 2.3.0
```



Let's try running that. Make sure your current working directory (`pwd`) is the flight-school folder and the build.yml file is located there.



```sh
$ fly -t meetup execute -c build.yml
executing build 107418
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 40960    0 40960    0     0   7657      0 --:--:--  0:00:05 --:--:--  9884
initializing
Pulling ruby@sha256:7ee3b80a4cca2d0545ca8a1cec01a880279a141bc0eb9e35ef192edd27652a8b...
sha256:7ee3b80a4cca2d0545ca8a1cec01a880279a141bc0eb9e35ef192edd27652a8b: Pulling from library/ruby
7268d8f794c4: Pulling fs layer
[... docker output ...]
b3789cc88ffa: Pull complete
Digest: sha256:7ee3b80a4cca2d0545ca8a1cec01a880279a141bc0eb9e35ef192edd27652a8b
Status: Downloaded newer image for ruby@sha256:7ee3b80a4cca2d0545ca8a1cec01a880279a141bc0eb9e35ef192edd27652a8b

Successfully pulled ruby@sha256:7ee3b80a4cca2d0545ca8a1cec01a880279a141bc0eb9e35ef192edd27652a8b.

running flight-school/ci/test.sh
+ pushd flight-school
/tmp/build/e55deab7/flight-school /tmp/build/e55deab7
+ bundle install
Fetching gem metadata from https://rubygems.org/..........
Fetching version metadata from https://rubygems.org/..
Installing addressable 2.4.0
Installing safe_yaml 1.0.4
Installing diff-lcs 1.2.5
Installing hashdiff 0.3.0
Installing rack 1.6.4
Installing rspec-support 3.4.1
Installing tilt 2.0.2
Using bundler 1.11.2
Installing crack 0.4.3
Installing rack-protection 1.5.3
Installing rack-test 0.6.3
Installing rspec-core 3.4.3
Installing rspec-expectations 3.4.0
Installing rspec-mocks 3.4.1
Installing webmock 1.24.0
Installing sinatra 1.4.7
Installing rspec 3.4.0
Bundle complete! 4 Gemfile dependencies, 17 gems now installed.
Bundled gems are installed into /usr/local/bundle.
+ bundle exec rspec
Run options: include :focus=>true

All examples were filtered out; ignoring :focus=>true

Randomized with seed 20539
........

Finished in 0.39713 seconds (files took 0.20659 seconds to load)
8 examples, 0 failures

Randomized with seed 20539

+ popd
/tmp/build/e55deab7
succeeded
```



Woohoo! We've run our unit tests inside Concourse. Now is a good time to
commit and push.

In general, try and think in terms of small reusable tasks that perform
a simple action with the inputs that they're given. If a task ends up
having too many inputs then it may be a smell that your task is doing
too much. Similar to if a function in a program you were writing had a
lot of parameters. In fact, that's a good way to think about tasks:
they're functions that take inputs as parameters. Keeping them small and
simple allows you to easily run them from your local machine as above.

Please excuse the long-winded iterative process we used to get to the
final result. You'll end up writing enough tasks that you can skip
directly to the end. We felt it was important to go through all the
possible hurdles you may encounter on your journey.




## Starting a Pipeline

Ok, so. We have a task we can run. How about we run that every time the
code changes so that we can check to see when anything breaks. Enter:
pipelines.

Pipelines are built up from resources and jobs. Resources are external,
versioned things such as Git repositories or S3 buckets and jobs are a
grouping of resources and tasks that actually do the work in the system.

Save the following to a `ci/pipeline.yml` file.

```yaml
resources:
- name: flight-school
  type: git
  source:
    uri: https://github.com/(your username)/flight-school
    branch: master

jobs:
- name: test-app
  plan:
  - get: flight-school
  - task: tests
    file: flight-school/build.yml
```

Since the pipeline wants to execute our new `build.yml` file which in turn runs `ci/test.sh` we will need to add, commit and push these files to our repository:

```sh 
git add .
git commit -m 'Add build task'
git push origin master
```

Now we are ready to *upload* our pipeline to Concourse:


```sh
fly -t meetup set-pipeline -p flight-school -c ci/pipeline.yml
...
pipeline created!
you can view your pipeline here: https://(your concourse url)/pipelines/flight-school

the pipeline is currently paused. to unpause, either:
  - run the unpause-pipeline command
  - click play next to the pipeline in the web ui
```



Follow the instructions to unpause the pipeline.

Click the job. Then click run.

It runs!


### Running Continuously

Add `trigger: true` to the `get`. Any new versions will trigger the job.

```yaml
jobs:
- name: test-app
  plan:
  - get: flight-school
    trigger: true
  - task: tests
    file: flight-school/build.yml
```

Try pushing another commit to the repository. For extra credit push a commit
that breaks the build and then another one that fixes it again.
