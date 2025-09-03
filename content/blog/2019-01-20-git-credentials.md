+++
title = 'Store git credentials for remote servers'
date = '2019-01-20'
slug = "store-git-credentials-for-remote-servers"
+++

You're taking a CS course at UBC and just finished a chunk of work that you wanna commit.

When you try to `git push` or  `git pull`, you're asked to type in username and password:

![ex_git_push](https://user-images.githubusercontent.com/9669739/51456134-5df4ae00-1d01-11e9-91e6-d0b83e013f24.png)

This guide will show how you can use git config to cache your authentification info

## Saving credentials

git have this nice feature called [`credentials`](https://git-scm.com/docs/gitcredentials) that let you store authentication credentials for git servers (like the `stash.ugrad.cs.ubc.ca` for UBC)

We'll be using `cache` option which will store the credentials temporarily.

Run the following command to enable this git option:

```sh
# 10368000 is 4 months in seconds
# With this config, git will remove credential from memory after 4 months
git config --global credential.helper 'cache --timeout=10368000'
```

You can check that this config is enabled by running the following command:

```sh
# Should return output like this
# [credentials]
#        helper = cache --timeout=10368000
cat ~/.gitconfig
```

## Saving credentials permanently

If you want to save the password permanently in the system, you can use `store` option like in the following command:

```sh
git config --global credential.helper store
```

However note that once you type credentials for git server, the credentials will be stored in `~/.git-credentials` file as **plain text file**!

Hope it helps!
