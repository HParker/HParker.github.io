Orbtoberfest by CircleCI 2019

![orbtober](/orbtober.png)

CircleCI put on an event in Seattle to promote their open [Orb registry](https://circleci.com/orbs/registry/). Orbs are their shared configuration format that allows you to write a CircleCI configuration for a job that anyone can use. This was possible before Orbs by sharing a bit of YAML that you could copy into your existing CircleCI config, but now you can reference the Orb by name and leave the rest up to the Orb. They call it [Orbtoberfest](https://hacktoberfest.circleci.com) which is kinda cute.

The simplest Orb YAML file looks like:
```yml
version: 2.1
description: Hello World Orb

commands:
  hello-world:
    description: say hello world
    steps:
    - run:
        command: echo Hello, World
        name: hello-world

examples:
  hello-world:
    description: Say hello
    usage:
      orbs:
        hello-world: hparker/hello-world@x.y.z
      version: 2.1
      workflows:
        test:
          jobs:
          - hello-world/hello-world
        version: 2

executors:
  ruby:
    docker:
    - image: circleci/ruby:2.4.2-jessie-node
jobs:
  hello-world:
    description: Say hello world
    executor: << parameters.executor >>
    parameters:
      executor:
        default: ruby
        description: executor for hello world
        type: executor
    steps:
    - hello-world

```

Someone can add this to their existing config with,

```yml
orbs:
  hello-world: hparker/hello-world@0.0.1

version: 2.1

test:
    docker:
      - image: rubylang/ruby:latest

workflows:
  test:
    jobs:
    - hello-world/hello-world
  version: 2
```

You can save arbitrarily large and complicated bits of configuration from your CircleCI

## Surprisingly helpful

During the event, I wanted to release an Orbs for some of the popular ruby linting libraries. While working on that, I made a mistake. I made my config say,

```yml
jobs:
  reek:
    description: Checkout code and run Reek
    executor: << parameters.executor >>
    parameters:
      executor:
        default: ruby
        description: executor for reek
        type: executor
      options:
        default:
        description: CLI options
        type: string
      version:
        default: ""
        description: Reek version
        type: string
```

Under `options:` I left the default line empty. I thought this would add an empty string default, which it doesn't. This makes the field required. So now, if I call the `reek` Orb from my project's config,

```yml
orbs:
  hello-world: hparker/reek@0.0.1

version: 2.1

test:
    docker:
      - image: rubylang/ruby:latest

workflows:
  test:
    jobs:
    - reek/reek
  version: 2
```

validating my circle config will actually tell me that I didn't pass the required argument!

```bash
$ CircleCI config validate .circleci/config.yml
Error: Error calling workflow: 'test'
Error calling job: 'reek/reek'
Missing required argument(s): options
```

This is a bit of effort put into making Orbs easier to use that I really appreciate. I assumed that validating your config would just check that it is syntactically correct, but it actually knows the contract provided by the Orb. I was pleasantly surprised that CircleCI put in the extra effort to make that work.

## Why are Orbs cool?

I am actually excited for more projects to adopt Orbs. Orbs allow open source tools to more easily provide a service equivalent to closed source alternatives. Now instead of paying someone to run a linter and code analysis tools over your code, the open source project can provide an Orb to run and integrate with your CI as easily as closed source solutions.

You can see the Orbs I created on the registry under [hparker](https://circleci.com/orbs/registry/?query=hparker&filterBy=all) Please feel free to use them in your projects and contribute changes you might want.

Huge thanks to the author of the RuboCop Orb which I took most of my inspiration from.