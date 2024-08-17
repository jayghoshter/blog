+++ 
date = "2023-02-17"
title = "direnv and mamba for python dev shells"
description = ""
slug = "direnv-mamba-python-dev-shells" 
tags = []
categories = []
externalLink = ""
series = []
+++

I am a proponent of Nix for stable and reproducible builds of software. Theoretically, it's all positives: 

- symlink-to-store based environment to allow multiple software (dependency) versions live conflict-free, 
- allow impromptu overrides and custom builds of any package/dependency
- cool `nix-shell` (or `nix develop`) feature for hacking away at your software 
- ... and so much more that I don't care to research and write about.

But as anyone but the most hardcore Nix devs can attest to, the actual experience of using Nix is quite frustrating: 

- A hard time getting to work with python. I don't care if it's the fault of the python ecosystem. The tool doesn't do what it is supposed to do as easily as it says it would. Especially when it comes to support for bleeding edge PyPI packages and/or python versions. And don't give me any spiel about `mach-nix`, `poetry2nix` or `pypi2nix`. These things are patchwork fixes to a broken system in my opinion. They might work in specific cases, but are nowhere close to robust.
- If there isn't a nixpkg for what you want, it's probably gonna make you wait while it builds from source.
- If you end up on an old-ish machine without root access and kernel user namespaces, it might not even be possible to run Nix.
- To flake or not to flake?
- Documentation? Where are you?
- ... and so much more that I don't care to research and write about.

I think the major disappointment is that you believe in the promise of a unified reproducible package manager/build system and you scour through someone's years old, deprecated diary of experimenting with Nix, trying to figure it out so that you can reach dev-nirvana, all for seemingly  nothing. It's not as perfect as the evangelists tout even though it aspires to be.

So it is with a heavy heart that I must resort to using `mamba` + `direnv` for my Python projects. Here's my `.envrc`.

```
#!/bin/bash

# -------
# Globals
# -------
MINICONDA3_PATH="/path/to/miniconda3"
ENV_NAME=$(basename $PWD)
PYTHON_VERSION=3.11

# -----------------
# Utility functions
# -----------------
msg() {
  echo >&2 -e "${1-}"
}

die() {
  local msg=$1
  local code=${2-1} # default exit status 1
  msg "$msg"
  exit "$code"
}

# shortcut for creating new conda environments
mamba_new_env() {
    mamba create -n $1 -y python=$PYTHON_VERSION
    mamba activate $1
    if [ -f "./requirements.txt" ]; then
        mamba install -c conda-forge -y --file requirements.txt
        if [[ "$?" == 0 ]]; then
            echo "Mamba failed. Trying pip..."
            pip install -r requirements.txt
        fi
    fi
}


# --------------------------
# Conda/Mamba initialization
# --------------------------
SHELLNAME=$(basename $SHELL)
[[ "$SHELLNAME" == "bash" ]] || [[ "$SHELLNAME" == "zsh" ]] || die "unknown shell: $SHELL"

__conda_setup="$( $MINICONDA3_PATH/bin/conda 'shell.'$SHELLNAME 'hook' 2> /dev/null)"
if [ $? -eq 0 ]; then
    eval "$__conda_setup"
else
    if [ -f "$MINICONDA3_PATH/etc/profile.d/conda.sh" ]; then
        . "$MINICONDA3_PATH/etc/profile.d/conda.sh"
    else
        export PATH="$MINICONDA3_PATH/bin:$PATH"
    fi
fi
unset __conda_setup

if [ -f "$MINICONDA3_PATH/etc/profile.d/mamba.sh" ]; then
    . "$MINICONDA3_PATH/etc/profile.d/mamba.sh"
fi

# ----------------
# Environment code
# ----------------
# Activate or create mamba environment
mamba activate $ENV_NAME || mamba_new_env $ENV_NAME

# Export Env Vars
export PYTHONPATH=$(pwd):$PYTHONPATH
export PATH=$(pwd)/bin:$PATH

# Additional Code
```

It isn't as ideal as a perfect Nix setup could have been, but it's good enough to get the job done. It's fast and sane and it lets me get to work. 

With Nix, not all python packages are directly available. Using mach-nix, you add an additional layer of problems: I, personally, couldn't get it running with Python3.11. Python3.9 works directly since it's the current default, and 3.10 can work with some overrides. But even then, if you want a package that was uploaded to PyPI in the last few hours (say, you were waiting for some crucial bug fix), you'd have to wrestle with some annoying `pypi-deps-db` stuff.

I don't currently have the time to contribute to nixpkgs. Although one of these weekends, I'd like to sit down and take a look.
