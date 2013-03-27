---
layout: post
title: "Deploying a Ruby app in an hostile environment"
---

\[Disclaimer: Here is a blog-post I wrote for my company blog. You can find the
original text at
<http://www.mentel.com/deploying-a-ruby-app-in-an-hostile-environment/>. They
kindly allow me to publish a copy on here!\]

While the developer's job is to design and create the web applications the
client (internal or external) ordered and, most of the time, paid for, the
deployer's main task is basically "Don't screw things up! ... And don't let
anybody else do it either!"...

While the first is usually easy enough to do in a controlled environment once
you're used to it, I found the second to be frequently forgotten by colleagues
even in big/Fortune 500/worldwide corporations or governmental
agencies. Ironically enough, common sense with just the right dose of your usual
sysadmin paranoia is going to fix both these points and make your life much
easier!

Let's take the example of our own hosting infrastructure here at
Mentel. Although we are mostly hosting RoR applications, these could be easily
applied to other applicative solutions such as PHP websites. They are developed
internally or, sometimes, customer provided. While most sysadmins would consider
this a friendly environment and assume no application will ever be a threat to
the infrastructure, we prefer to use another approach. Our infrastructure is
completely built from a mildly hostile environment point of view.

From a general design POV, our servers are all protected by a centralized
firewall shielding them from both the evil outside world (aka Internet) and each
other. Each application uses strictly separated privileges and environments for
each of their 3-tiers access (file system, web server and database). Furthermore,
the permissions are restricted enough that no application could read others'
non-public data and, in no case could any application write data it doesn't own
itself.

Even the deployment steps themselves are done using separated access for each
application. That way no human error could ever crash another site than the one
we are currently working on. To further avoid human error during deployment, a
fully automatic deployment procedure has been developed so the infrastructure
could (and will, once the final touch has been applied), run using a DevOps
model.

Surprisingly enough, from a sysadmin point-of-view, this is easy to do. First,
the centralized firewall will be provided by any middle-to-high end
router. Worst case, you could just dedicate a small box as a gateway (it could
also be used as a VM host if you're hardware restricted). Second, privilege
separation is (Err... should be) a given for any hosting design: each
application gets its own DBMS user and DB (or schema) on which it is granted R/W
DDL and TCL access (eventually DML access if needed)
(<http://www.orafaq.com/faq/what_are_the_difference_between_ddl_dml_and_dcl_commands>).
On each server, the applications are each granted its own system user, which
will run the application server (Unicorn in our case). Access to the server for
the deployments is then provided via the deployer's public SSH key as said
user. How to centralize authentication will be part of another post, but in this
case, let's just say that although `sshd` is a bit irascible regarding symlinked
authorized_keys, it doesn't mind hard links, even if the file is owned by
someone else other than the connecting user (wink, wink; nudge,
nudge). Sysadmins on the other hand are each granted their own sudoer personnel
access. A default umask of 0067 and a www-data group will take care of file
system permissions. Of course, you then have to make the unicorn socket (this
one actually need to be writable too) and static assets group-readable so the
web server (nginx in our case) can read them directly.

All these ensure that no human error or willingly compromised site could impact
any other. These protections are required not because we don't trust developers,
but because "Sh\*t happens!": bugs exist and while it is no one's fault if some
make their way to PROD, it is everybody's if they impact more than they could
have. Moreover, Murphy's Law is never far from that small outdated framework you
deployed three years ago and moved on...

The funny part is, of course, when you want to automate the process. With Ruby,
you have two main deployment tools:
[Capistrano](https://github.com/capistrano/capistrano/wiki) and
[Vlad](http://rubyhitsquad.com/Vlad_the_Deployer.html). Both have their
internals equally (un)documented and include mildly defensive programming, but
Vlad has a smaller and simpler code base (easier to read, understand and patch),
is fully integrated with Rake (whereas Capistrano defines its own DSL "based" on
`Rakefile` syntax) and uses the system ssh (whereas Capistrano uses the net-ssh
and friends gems which does not support an as wide range of options and
features). To provide a highly separated environment, we use RVM and bundler.

Unfortunately, until recently, none of these tools were planned to operate in
such a restrictive environment (especially the separated users and restricted
umask/permissions issues). RVM was not playing well with the permissions and was
stubborn when switching gemsets or even at load time, preventing the environment
separation. Vlad and its modules has been at times less than bright regarding
permissions, and loading its small friends into the deployment environment or
runtime.  I am very grateful to [Michal Papis](https://github.com/mpapis) (RVM),
[Ryan Davis](https://github.com/zenspider) (rake-remote_task and Vlad), [Kevin
Bullock](https://github.com/krbullock) (vlad-unicorn) and [Dennis
Reimann](https://github.com/dennisreimann) (vlad-extras) for their patience and
trust regarding my clumsy PRs on GitHub. After a couple of months of their
collaboration and help, everybody is finally able to deploying all their RoR
applications in a seamless and robust fashion, even is the most paranoiac
environment, and you need to only slightly modify 4 files: `Gemfile`,
`Rakefile`, `deploy.rb` and `unicorn.rb`!

The last piece that was needed for our development environment, is to be able to
deploy from our internal private repositories. This means: ssh agent forwarding
and reverse port forwarding. Added to the one-application-one-user policy, it
made the ssh config for each project a bit heavy. The solution was to use two
`.ssh/config` for each project: One on the deployment side, referencing each
server (QA and PROD) with its options (HostName, User, RemoteForward,
ForwardAgent) and one on the remote server aliasing the `git` ssh hostname to
the local SSH Tunnel to our GitLab Host. This makes the config for each project
much easier and provides the access to the server with a unified `ssh -F
${PROJECT_ROOT}/.sshrc` prod which is the best for your ease of mind (Bonus
point: The "which server is actually this project on, again?" is easily answered
by a single `cat` command!).

Using this solution, we now are able to deploy ourselves, or even let our
customers deploy directly on our architecture without any worry regarding
contamination. Even if a backdoor was willingly upload in our environment, its
impact would be very limited to what an unprivileged standard user could do and
would be easily detected since each user is supposed to run its own Unicorn
server, in particular none of the other websites would be put at any risk
whatsoever.
