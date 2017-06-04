+++
categories = ["blog"]
date = "2017-06-04T07:49:16+01:00"
draft = false
title = "Setting Up Git and Vim For Chef"

+++

# Overview
Recently I have started writing Chef cookbooks again. Chef is an immensely powerful tool for configuration management and infrastructure automation. You can find more information about Chef [here](https://www.chef.io/chef/).

My text editor of choice is [Vim](http://www.vim.org). The main reason is that is always available on pretty much any Unix based OS, I love the modular nature of it and there is a level of mysticism to it that I secretly love (I feel it makes me a bit more legit!).

This article will explain how I set up linting and static analysis with the tools that come with the Chef DK ([foodcritic](http://www.foodcritic.io/) and [cookstyle](https://github.com/chef/cookstyle)), the [overcommit gem](https://github.com/brigade/overcommit) and [Syntastic](https://github.com/vim-syntastic/syntastic)

# Getting Set Up

## Chef DK
Install the Chef DK from [here](https://downloads.chef.io/chefdk). Chef have packaged this up nicely for the most common architectures.

## Vim Linter
A good practice when writing any code is to have a good [linter](https://stackoverflow.com/questions/8503559/what-is-linting) to help you catch any spelling mistakes, bad syntax etc. If using Vim then the best linter I have found is [Syntastic](https://github.com/vim-syntastic/syntastic). Installing it is super simple. I use [Pathogen](https://github.com/tpope/vim-pathogen) as package manager for Vim. The install process is as follows:

```
# Install Pathogen
mkdir -p ~/.vim/autoload ~/.vim/bundle && \
curl -LSso ~/.vim/autoload/pathogen.vim https://tpo.pe/pathogen.vim

# Enable Pathogen in .vimrc
echo "execute pathogen#infect()" >> $HOME/.vimrc

# Install Syntastic
cd ~/.vim/bundle && \
git clone --depth=1 https://github.com/vim-syntastic/syntastic.git
```

Now we have that sorted we need to enable a couple of extra settings in our `.vimrc`. These are settings that I find useful, feel free to check out [Syntastic's Docs for other options](https://github.com/vim-syntastic/syntastic#installpathogen)

```vim
" General
set statusline+=%#warningmsg#
set statusline+=%{SyntasticStatuslineFlag()}
set statusline+=%*

let g:syntastic_always_populate_loc_list = 1
let g:syntastic_auto_loc_list = 1
let g:syntastic_loc_list_height = 3 
let g:syntastic_check_on_open = 1
let g:syntastic_check_on_wq = 0

" Language Specific
" HTML
let g:syntastic_html_tidy_exec = 'tidy5' " use tidy5 html linter
" JS
let g:syntastic_javascript_checkers = 'eslint' " use standard js linter
" Golang
let g:syntastic_go_checkers = ['golint', 'govet', 'errcheck']
let g:syntastic_mode_map = { 'mode': 'active', 'passive_filetypes': ['go'] }
let g:go_highlight_functions = 1
let g:go_highlight_methods = 1
let g:go_highlight_fields = 1
let g:go_highlight_types = 1
let g:go_highlight_operators = 1
" Ruby
let g:syntastic_ruby_checkers = 'rubocop'
let g:syntastic_ruby_rubocop_exec = '/usr/local/bin/cookstyle'
```

## Overcommit
Every piece of development should be under version control. I use git for version control and so should you :). Static code analysis, code style, linting etc is something that I always seem to forget so to help with that there is a tool that uses Git hooks to run linters etc before you are able to finish a commit. The tool is called [overcommit](https://github.com/brigade/overcommit). Here are steps to install:

```
gem install overcommit        # install overcommit
cd <chef repo>
overcommit --install          # install hooks to repo
vi .overcommit.yml            # add checks
overcommit --sign             # update overcommit
```

# In Editor Linting
Chef cookbooks lay out the desired state for a specific service and they are written in plain ol' Ruby.

This means that we are able to use [Rubocop](https://github.com/bbatsov/rubocop) as our linter of choice. Rubocop has beceom the defacto linter and static code analyser for Ruby code.

For Chef cookbooks Chef has created a standard set of configuration files for Rubocop and packaged it up into a tool called [cookstyle](https://github.com/chef/cookstyle). 

You will need to replace the default rubocop linter with cookstyle in your `.vimrc`. At the moment I haven't found a good way to automatically switch between cookstyle and rubocop so that for normal Ruby use Rubocop and Chef files use Cookstyle. Either way here is the code to do this:

```vim
let g:syntastic_ruby_checkers = 'rubocop'
let g:syntastic_ruby_rubocop_exec = '/usr/local/bin/cookstyle'
```

What we're doing here is simply replacing the executable path for rubocop to be cookstyle.

# Overcommit Config

Now we want to set our overcommit.yml to be useful. Below is the config that I have been using and has been quite successful so far.

```
gemfile: Gemfile

PreCommit:
  Foodcritic:
    enabled: true
    include:
      - 'attributes/**/*'
      - 'definitions/**/*'
      - 'files/**/*'
      - 'libraries/**/*'
      - 'providers/**/*'
      - 'recipes/**/*'
      - 'resources/**/*'
      - 'templates/**/*'

	HardTabs:
		enabled: true

	MasterHooksMatch:
		enabled: true
		quiet: true

	RuboCop:
		enabled: true
		include:
			- '**/*.rb'
			- 'Gemfile'

	TrailingWhitespace:
		enabled: true

	YamlSyntax:
		enabled: true
```

Now whenever I make a commit overcommits git hooks will run through and check if any of the specified linters have failed like so:
```
git add .
git commit -m 'Commit message'
Running pre-commit hooks
Check YAML syntax........................................[YamlSyntax] OK
Check for trailing whitespace....................[TrailingWhitespace] OK
Analyze with Foodcritic..................................[Foodcritic] OK

✓ All pre-commit hooks passed

Running commit-msg hooks
Check for trailing periods in subject................[TrailingPeriod] OK
Check text width..........................................[TextWidth] OK
Check subject capitalization.....................[CapitalizedSubject] OK
Check subject line................................[SingleLineSubject] OK

✓ All commit-msg hooks passed

[master adbeba1] Commiting
 Date: Sun Jun 4 09:13:44 2017 +0100
 3 files changed, 6 insertions(+), 6 deletions(-)

```

# Summary
And that's about it. With the relevant linters and analysers and bit of automation using git hooks with Overcommit we should now be able to code faster, with less errors and more reliability.
