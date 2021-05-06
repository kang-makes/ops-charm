# ops-charm

Is was automation tool for environment management.

It allowed to isolate a group of variables fron own environment to another
and initialize an GPG ssh agent, gopass agent automatically if some variables
and files exists.

This was overengineer and can be managed by a proper use of GPG keys mixed with some git magic like identity providing:

> .config/git/config
```toml
[user]
	useConfigOnly = true

[alias]
	identity = "! git config user.name \"$(git config user.$1.name)\"; git config user.email \"$(git config user.$1.email)\"; git config user.signingkey \"$(git config user.$1.signingkey)\"; :"
	identity-list = "!identities() { git config --list --name-only | grep -E 'user\\..*\\.signingkey' | sed -e 's/^user\\.\\(.*\\)\\.signingkey/\\1/'; }; identities"

[user "company"]
	email = kang@a-company.es
	name = A more serious name
	signingkey = A SIGNING KEY

[user "personal"]
	email = devel@my-email.es
	name = My name in github
	signingkey = OTHER SIGNING KEY
```

Here no repository will have user and mail and a valid `git identity` has to be inserted for each repository.

Environment variables could be sourced with dotenv files automatically by a (shell plugin)[https://github.com/ohmyzsh/ohmyzsh/blob/master/plugins/dotenv/dotenv.plugin.zsh]

About mantaining developer environments its is prefered to use `poetry` or `pipenv` for python instead my approach or simply use the environment you need using `nix`.

# Old README

## How it works and isolates environments.

ops-charm simply uses `virtualenv_wrapper` to which from an environment to another.
It uses `virtualenv_wrapper` scripts so can add some capabilities that all my environments
use like having a shared set of keys for autentication, a team common key vault using gopass,
an environment for ansible and its dependencies, etc.

So you use `workon` to change to an environment (as with a virtualenv_wrapper environment) and


In an environment there are 4 folders:

```
bin
bin-global
functions
functions-global
```

`bin` and `functions` folders are used only while you have activated the environment while
`bin-global` and `functions-global` are used on any shell.

`functions-global` is used for loading ops-charm itself.

Evey ops-charm environment has to have a `ops-charm.values` which will be sourced when
activating an environment.

### Configure your environments path

Set something like this on your `.bashrc`.

```bash
export ENVIRONMENTS="$HOME/envs"

for bash_function_file in $ENVIRONMENTS/*/functions-global/*; do
  if [ -f "$bash_function_file" ]; then
    source $bash_function_file
  fi
done

for executables_folder in $HOME/go/bin $HOME/.local/bin $ENVIRONMENTS/*/bin-global; do
  if [ -d "$executables_folder" ]; then
    export PATH=$executables_folder:$PATH
  fi
done
```

## Prerequisites

Install this dependencies from your repos:

```
virtualenv_wrapper
gpg     # Not needed but recommended
gopass  # Not needed but recommended
chezmoi # Not needed but recommended
```


### GPG

It is used as a ssh agent and for crypting gopass passwords.

For the agent to be activated `GNUPGHOME` environment variable and a file in
`$GNUPGHOME/sshcontrol` must exist.

`GNUPGHOME` could point to a path that is outside of the environment so the kayring
could be used by more than one environment. For example having and environment for the
team itself with the gopass passwords ans gpg keyring and 3 environments for production,
staging and devel which GPG and gopass variables point to the team environment.


### gopass

Is is used for store team passwords securely.


For the gopass agent to be activated `GOPASS_CONFIG` environment variable and a file in
that path sould exist.

> NOTE: if you use the keys subdirectory gopass will fail with strange errors of an
> unexisting repo (because could not find de `.git` folder) as it does not use git
> CLI for testing if the folder is a repo on a higher folder on the tree.
>
> If this happens, `autosync` has to be disabled from `gopass` and `path` in `config.yaml`
> should be changed to something like `path: gpgcli-noop-fs+file://`.

### Chezmoi

Chezmoi is a tool for version-control all the `dotfile` you want to control.

A good use of this is to keep on your environment or your team all your dotfiles so
all the team has the same configuration and/or aliases.

chezmoi expects, as same a gopass, to have a repo for itself but opposed to gopass
chezmoi uses git cli for everything so you can use chezmoi in a subfolder without any problem.

> NOTE: the only problem with chezmoi in a subfolder is on `init` stage if the repo has a
> `.chezmoi.toml.tmpl` in its root so every `chezmoi init` could result in clning your repo in
> `$HOME/evns/template/chezmoi` subfoler instead of `$HOME/evns/template` so ops-charm environment
> and chezmoi end unusable withou manual intervention. Moving manualliy the folder to its father or
> `rm ~/.config/chezmoi/chezmoi.toml && chezmoi init --source ~/envs/template github.com/you/template`
