---
layout: page
title: "Cloud Development Environment: The Definitive Guide"
permalink: /cloud-development-environment-the-definitive-guide/
---

Cloud development environment is not a new concept.
It has been around for a while.
However, there are still many tricky parts that keep people from adopting it.
We already adopted infrastructure as code, can we apply the same concept to
development environment?

In this post, I will show you how to set up a flexible and powerful cloud
development environment. In the end, we will achieve the ultimate goal:

**Develop Any Code, Anywhere, Anytime**

I will cover a few tools and techniques, including

  * [Vagrant](https://www.vagrantup.com/)
  * [Docker](https://www.docker.com/)
  * [VS Code Remote Development](https://code.visualstudio.com/docs/remote/remote-overview)
  * [JetBrains Remote Development](https://www.jetbrains.com/remote-development/)
  * [Development Containers](https://containers.dev/)
  * [GitHub Codespaces](https://github.com/features/codespaces)

### Vagrant

[Vagrant](https://www.vagrantup.com/) is a tool for building and managing
virtual machine environments. It supports multiple providers,
the most popular one is VirtualBox. The flow of using Vagrant is:

  1. Write a `Vagrantfile` to describe the virtual machine.
  2. Run `vagrant up` to create the virtual machine.
  3. Run `vagrant ssh` to login to the virtual machine.

All environment setup and configuration are done in the `Vagrantfile`.
It can also mount host directories to the virtual machine, so that we can
edit files on the host and run them in the virtual machine.

But there is a problem with that, we want to put all dependencies in the
virtual machine, so that we don't need to install them on the host.
However, most IDE/editors on host require those dependencies to provide better
support such as auto-completion, refactor, running tests, etc.

In the past, we either need to install dependencies on the host,
or use an editor inside the virtual machine via ssh such as vim.
There are some new tools that can help us solve this problem.
We will cover them later.

Another good thing about Vagrant is the virtual machine (box) can be shared
with others, so we can easily set up the same environment for other
developers. There are many public boxes available on
[Vagrant Cloud](https://app.vagrantup.com/boxes/search).

### Docker

[Docker](https://www.docker.com/) is mainly used for building and running
containerized applications. But with proper setup, it can also be used as a
development environment. You may know how Docker solved
`It works on my machine` problem.
By developing in the same container as production, we can guarantee that
if it works on my machine, it will work in production as well.

The flow of using Docker is:

  1. Write a `Dockerfile` to describe the container.
  2. Run `docker build` to build the container.
  3. Run `docker run` to run the container.

After code changes, we can rebuild the container and run it again.
If the container is running in Kubernetes,
we can use [skaffold](https://skaffold.dev/) to automate the process.

The problem with this approach is similar to Vagrant.
We can't use the IDE/editor without installing dependencies on the host.
If it's not done properly, it can cause a lot of troubles such as it works
on the host but not in the container, or vice versa.

### VS Code Remote Development
[VS Code Remote Development](https://code.visualstudio.com/docs/remote/remote-overview)
is a game changer for cloud development environment. It will start a remote
code server to handle all heavy work such as auto-completion, refactor,
running tests, etc. The client side just acts as a thin UI layer.
It supports multiple ways of connection, including SSH, Container and WSL.

The flow of using VS Code Remote Development is:

  1. Prepare remote environment (Vagrant, Docker or WSL).
  2. Remote connect to the environment using `Remote Development` extension.
  3. Edit code or run commands in the remote environment.

By using VS Code Remote Development, we only need to have VS Code installed on
host, all other dependencies are installed in the remote environment.

There are many handy features in VS Code Remote Development, such as auto
detect port forwarding. You can even setup a
[code server](https://github.com/coder/code-server) on a remote server, then
edit code in the browser.

### JetBrains Remote Development

[JetBrains Remote Development](https://www.jetbrains.com/remote-development/)
is very similar to VS Code Remote Development. It also starts a remote IDE
backend to handle all the processing. IDE on host talks to the backend via
[Gateway](https://www.jetbrains.com/remote-development/gateway/).

The biggest advantage of JetBrains Remote Development is that it has powerful
refactor support, same experience as local development when network is good.

### Development Containers

[Development Containers](https://containers.dev/) is a new concept introduced
by VS Code. The basic idea is to use a container as development environment.
It's not exactly the same as production environment, it contains many
additional tools to provide better developer experience, for example,
`docker from docker` let you use host docker inside the container.

There are many base images available on
[Development Container Images](https://hub.docker.com/_/microsoft-vscode-devcontainers).

The flow of using Development Containers is:

  1. Write a `Dockerfile` to describe the container.
  2. Write a `devcontainer.json` to customize the container.
  3. Use VS Code `Remote-Containers` extension to open the code.

The rest of the flow is the same as VS Code Remote Development.

It can also be used apart from VS Code.
[Dev Container CLI](https://github.com/devcontainers/cli) can be used to manage
dev containers like `docker` command. For example, `devcontainer up` can create
and run dev container, if there is sshd running in the container, you can use
other remote development tools (like JetBrains Remote Development) to connect
to it.

If the development environment is complex, you can use multiple containers
together, it has
[docker compose support](https://code.visualstudio.com/remote/advancedcontainers/connect-multiple-containers).

### GitHub Codespaces

[GitHub Codespaces](https://github.com/features/codespaces) is a new service
on GitHub that provides cloud development environment. It's based on Development
Containers, so it has all the features above. Additionally, it's running on
cloud machines, and has built-in VS Code Web UI.

It also has [dotfiles support](https://docs.github.com/en/codespaces/customizing-your-codespace/personalizing-github-codespaces-for-your-account),
so you can customize the environment by extracting your own dotfiles to a
repository, then Codespaces will automatically execute it when creating a new
environment.

### Combo

To get the best experience, we can combine multiple tools together.

For code hosted on GitHub, we can use GitHub Codespaces, setup `.devcontiner`,
and use VS Code or Browser to edit. If we want to use JetBrains IDE, we can
easily setup [ssh](https://cli.github.com/manual/gh_codespace_ssh), then use
JetBrains Remote Development to connect to it.

If develop inside the container is not enough, such as need to spin up other
virtual machines, or access some features only available on host, then we can
use Vagrant to create a virtual machine, and use VS Code Remote Development or
JetBrains Remote Development to connect to it.

### Conclusion

Cloud development environment can greatly boost developer productivity, and
improve developer experience. With the help of new tools like devcontainer,
remote development integrations in VS Code and JetBrains IDE, we can easily
start developing in a thin client device, leave the heavy work to the remote.
Now we can truly develop any code, anywhere, anytime.
