---
title: Adding non-tracked bundled gems
subtitle: "Posted: 2016-03-14 | Author: Victor Koronen"
layout: default
---

When collaborating on a Ruby project, you might run into a situation where you
would like to add gems to the project's gem bundle without adding it to
`Gemfile`, forcing the gems onto your collaborators. Still, you want to have
them managed by Bundler.

One such type of gem is an RSpec formatter, like [Fuubar][fuubar]. Another
example could be the [Guard][guard] suite of gems.

This is possible to accomplish without any changes to files tracked in the
project repository, by leveraging some Git and Bundler configuration directives.

[fuubar]: https://github.com/thekompanee/fuubar
[guard]: http://guardgem.org/

## Ignoring our local configuration

First of all, make sure that `.bundle/` and `Gemfile.local*` are ignored by Git.
This can be done on a global or on a repository level.

### Globally

Set up a [global `.gitignore` file][global-gitignore] and add the patterns to
it.

Run the following commands from anywhere.

    git config --global core.excludesfile ~/.gitignore
    echo "/.bundle/" >> ~/.gitignore
    echo "/Gemfile.local*" >> ~/.gitignore

[global-gitignore]: https://help.github.com/articles/ignoring-files/#create-a-global-gitignore

### Per repository

Add the patterns to the [non-tracked local `exclude` file][git-info-exclude].

Run the following commands from the project repository root.

    echo "/.bundle/" >> .git/info/exclude
    echo "/Gemfile.local*" >> .git/info/exclude

[git-info-exclude]: https://help.github.com/articles/ignoring-files/#explicit-repository-excludes

## Setting up our local configuration

1. Create `Gemfile.local`

       echo 'eval_gemfile "Gemfile"' > Gemfile.local
       echo 'gem "fuubar"' >> Gemfile.local
       echo 'gem "fuubar-cucumber"' >> Gemfile.local

2. Create `Gemfile.local.lock` (assuming the project has a `Gemfile.lock` file)

       cp Gemfile.lock Gemfile.local.lock

3. Tell Bundler to use `Gemfile.local`

       bundle config --local gemfile Gemfile.local

4. Install gems

       bundle install

5. Profit!

       bundle exec rspec -f Fuubar

## Caveats

One caveat with this setup is that [Bundler binstubs][bundler-binstubs] won't
work unless you set the `BUNDLE_GEMFILE` environment variable too. This is
because Bundler binstubs only read and conditionally set `BUNDLE_GEMFILE`. They
never read `.bundle/config`.

    export BUNDLE_GEMFILE=$(readlink -f Gemfile.local)
    bin/rspec -f Fuubar

Another caveat is that updates to `Gemfile.lock` and `Gemfile.local.lock` need
to be synced manually. If someone else updates `Gemfile.lock` those changes need
to be copied to `Gemfile.local.lock`. Similarly, if you update a gem, you need
to make sure the update is applied to both `Gemfile.lock` and
`Gemfile.local.lock`.

[bundler-binstubs]: https://github.com/rbenv/rbenv/wiki/Understanding-binstubs
