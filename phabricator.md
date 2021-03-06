


# Using Phabricator for GHC development



[
Phabricator](http://phabricator.org) (a.k.a."Phab") is a suite of tools for software development, initially developed by Facebook. The functionality is similar to using github and TravisCI: it is primarily used for code review and automated builds. Uses include:


- side-by-side diffs, inline comments, blocking states, command line access
- Better notifications: use "Herald" to control emails very closely, or commits or reviews you are interested in.
- Increased visibility for code review and feedback.
- Automatic `./validate` builds when using Arcanist! (see [Phabricator/Harbormaster](phabricator/harbormaster))
- [ Meme support](https://phabricator.haskell.org/macro/) (optional)
- Other features you might like, such as an IRC bot, and [
  Terminator jokes](https://github.com/phacility/phabricator/blob/8756d82cf6c13d86019efeb9df8bcdaad1b17ec8/src/infrastructure/daemon/bot/handler/PhabricatorBotObjectNameHandler.php#L66).

## Signing up



First off, sign up by going to the Phab instance at Haskell.org: [
https://phabricator.haskell.org/](https://phabricator.haskell.org/)



Next, click the power button in the top right corner. You'll be taken to a login screen where you can register a new account. You have several options, including a username/password, or using OAuth to login with GitHub, Facebook, or more.



Afterwords Phab will send you a verification email, which you'll need to open and read - so use a correct email address!


## Adding an SSH public key



Much like GitHub, you will need to add an SSH public key to your account to push Differentials. To add a key, go to your Phabricator [
settings page](https://phabricator.haskell.org/settings/), select the *Personal Account Settings* item, and then select the *SSH Public Keys* option from the list on the left. This will bring you to your SSH key list, to which you can add a new key by selecting the *Upload Public Key* option from the *SSH Key Actions* menu button in the top-right corner of the page. Here you can specify a name of your choice for the key, as well as the key contents (e.g. the contents of `$HOME/.ssh/id_rsa.pub`). Be sure to paste your *public* key (ending in `.pub`), not your private key.



Don't have an SSH key? Generate one with `ssh-keygen`.



If you use a specific SSH key for GHC or Haskell related work, you should specify that key for `*.haskell.org` because the phabricator `arc` tool doesn't push to the actual GHC repo, but to a staging repo on a different machine in the `haskell.org` domain.


## Understanding the interface



Phabricator is actually a suite of applications. The one you will use the most is **Differential, located on the top of the left-side panel**. 



You will also *really* want the command line tool, **Arcanist**. It's not strictly necessary, but makes code review and many other tasks significantly faster and easier.


## The CLI: Arcanist



The most important tool you're going to use with Phab is called **arcanist**, which is the command line interface to Phabricator. Arcanist is a tool written in PHP, and comes from a git repository - so you'll need PHP installed on your machine.



To install PHP on Debian or Ubuntu, run `apt-get install php-cli php-curl`.
On OS X PHP is already installed and you can skip directly to grabbing the arcanist sources.



**WARNING**: Don't install `arc` using the package manager from your Linux distribution (e.g. Debian, even unstable). It might work when you first install it from Debian, but sooner or later the packaged version will fall behind the version used on the server. You need the source checked out.



Getting the source to Arcanist is very easy, just check out two git repositories next to each other, and put the binaries in your `$PATH`:


```wiki
$ git clone https://github.com/phacility/libphutil.git
$ git clone https://github.com/phacility/arcanist.git
$ export PATH="$(pwd)/arcanist/bin:$PATH"
```


That's it! **This will even work on Windows, as long as you have PHP available in your shell**.



Next, make sure the `.arc-linters/arcanist-external-json-linter` submodule in the GHC source tree is initialized properly via `git submodule update --init`. Otherwise, you will get this [
scary error](https://mail.haskell.org/pipermail/ghc-devs/2016-May/012050.html):


>
>
> Usage Exception: Failed to load phutil library at location '\~/mycode/ghc/.arc-linters/arcanist-external-json-linter'. This library is specified by the "load" setting in ".arcconfig". Check that the setting is correct and the library is located in the right place.
>
>


Now, you're going to need to install a user certificate, so that `arc` can authenticate as you. First, go to your GHC repository, then run `arc install-certificate`. You need to be in the GHC repository so `arc` knows what URL to authenticate against:


```wiki
$ cd ~/mycode/ghc
$ arc install-certificate
```


Follow the directions it specifies - afterwards, your arcanist tool will be properly authenticated! Now you can submit reviews, paste things, etc etc.



**Remember:** In order to use our continuous integration service (Harbourmaster) you need to [
upload your SSH key](https://phabricator.haskell.org/settings/).



*Note*: once the certificate is installed, it will be written to `$HOME/.arcrc`, so it doesn't need to be installed again.



**Question**: How does `arc install-certificate` know what URL to use?
**Answer**: It's located in a file called `.arcconfig` in the GHC repository. This is why you have to `cd` there first.


##
Help! I'm getting a strange error when running `arc` that I didn't get yesterday



Probably GHC's Phabricator instance was updated and your `arc` client is now out of sync. Try `arc upgrade`.



For other problems, file a ticket in the [
GHC project](https://phabricator.haskell.org/project/view/2/) on Phabricator.


## Starting off: Fixing a bug, submitting a review



(This assumes you have installed Arcanist).



First off, let's say you have a bug you want to fix. Go to the Trac page for the bug, and make yourself as the owner.



Next, checkout a branch for the bug, and work on it:


```wiki
$ cd ~/code/ghc-head
$ git checkout -b fix-trac-1234
$ emacs ...
$ git commit -asm "compiler: fix trac issue #1234"
```


Once you're ready to submit it for review, it's pretty easy. Just run:


```wiki
$ arc diff HEAD~
```


which submits the `HEAD` commit to Phabricator and use your SSH public key to push a branch to the GHC Differentials [
repository](https://phabricator.haskell.org/diffusion/GHCDIFF/). If you have not added an SSH key you can skip pushing the branch with the `--skip-staging` flag.



After pushing your new Differential `arc` will respond with the revision number and a URL where you can view and discuss your change.



**NOTE**:  The above assumes you only want to send one commit to start with, which is the current tip of your branch.  In general you can send multiple commits by saying `arc diff XXX` where `XXX` is the the commit \*before\* the commits you want to send.  When you do this, `arc` will squash all the commits between `XXX` and `HEAD` into a single commit with the commit message you enter during `arc diff`.  You can see which commits would be squashed with `git log XXX..HEAD`.  Typically this is used when you have a series of modifications to the original commit in your repo, but you want them to be reviewed as a single diff.  If you want separate diffs, then used the "stacked diffs" workflow described below.



You may also modify the commit more later, by making new commits, and running `arc diff` again. This will just update the existing review:


```wiki
$ emacs ...
$ git commit -asm "fix bug in new feature"
$ arc diff # update existing review
```

## Working with multiple dependent diffs



When you have multiple commits with dependencies between them, we call them "stacked".  This section will describe how to work with stacked diffs in Phabricator.  (If the commits do not depend on each other, typically you want to create a branch of your repo for each one and do a separate `arc diff` for each one)



First, use a single git branch for your work.  Let's assume you have multiple commits on this branch, call them `A`, `B`, and `C`, and you want to get each of them reviewed separately on Phabricator.


```wiki
$ git rebase -i HEAD~3
```


(change the "3" to the number of commits in your stack).  In the editor that pops up, change each commit to "edit" instead of "pick", and exit the editor.  Then repeatedly:


```wiki
$ arc diff HEAD^
$ git rebase --continue
```


Until you have uploaded all your commits to Phab.  Now you have multiple diffs on Phab. These can be set as depending on each other on Phab by setting Parten/Child Revision accordingly.



Later you'll probably want to modify one or more of these diffs in response to reviews.  I do it like this:


- Commit the changes onto your branch as a new commit
- `git rebase -i HEAD~4`
- move your new commit after the one it should modify, change it to "fixup", and add `x arc diff` after it
- exit the editor, and enter a description into the editor when `arc diff` runs


Basically the idea is to use `git rebase -i` to repeatedly edit and `arc diff` commits in the stack.  You may also need to `arc diff` the commits further up the stack when they are rebased against changes lower in the stack.



When it's time to push, I normally `git push` the whole stack together.  If you don't have commit access, one of the maintainers will do it for you.


## Everything else



Phabricator is a powerful tool, and it's designed to be a productivity tool above all else. That means it has quite a lot of functionality.



If you want to learn more, check out the '[Extras](phabricator/extras)' page which contains more details on the available applications, and other assorted tips.


