---
title: Merging Ansible vaults in Git
subtitle: "Posted: 2017-07-07 | Author: Victor Koronen"
layout: default
---

While on the subject of [making Git and Ansible play nice together]({% post_url
2016-12-22-diffing-ansible-vaults-in-git %}), wouldn't it be great if there was
a way to automatically resolve simple merge-conflicts in these vault files? To
demonstrate the issue, let's create two conflicting branches.

    $ git checkout -b a master
    Switched to a new branch 'a'
    $ echo -e "---\nfirst: line\ntop: secret" > secrets.yml
    $ ansible-vault encrypt secrets.yml
    Encryption successful
    $ git commit -am "Add a first line"
    [a 4d84b62] Add a first line
     1 file changed, 6 insertions(+), 6 deletions(-)
     rewrite secrets.yml (93%)
    $ git checkout -b b master
    Switched to a new branch 'b'
    $ echo -e "---\ntop: secret\nlast: line" > secrets.yml
    $ ansible-vault encrypt secrets.yml
    Encryption successful
    $ git commit -am "Add a last line"
    [b 6294a80] Add a last line
     1 file changed, 6 insertions(+), 6 deletions(-)
     rewrite secrets.yml (93%)
    $ git merge --no-edit a
    Auto-merging secrets.yml
    CONFLICT (content): Merge conflict in secrets.yml
    Automatic merge failed; fix conflicts and then commit the result.

Bummer! Let's check out the `master` branch and start over.

    $ git merge --abort
    $ git checkout master

Similar to [the `textconv` option][git-diff-textconv] for diffing, Git allows us
to define a custom merge driver using [the `merge.*.driver`
option][git-merge-driver].

    $ git config --local merge.ansible-vault.driver "./ansible-vault-merge %O %A %B %L %P"
    $ git config --local merge.ansible-vault.name "Ansible Vault merge driver"
    $ echo "secrets.yml diff=ansible-vault merge=ansible-vault" > .gitattributes

Git expects it to behave as follows.

> The merge driver is expected to leave the result of the merge in the file
> named with `%A` by overwriting it, and exit with zero status if it managed to
> merge them cleanly, or non-zero if there were conflicts.

Git is usually pretty good at resolving most plain text conflicts, so let's
leverage that functionality. All we need to do is manage the encryption parts.

    $ cat > ansible-vault-merge
    #!/bin/bash

    set -e

    ancestor_version=$1
    current_version=$2
    other_version=$3
    conflict_marker_size=$4
    merged_result_pathname=$5

    ancestor_tempfile=$(mktemp tmp.XXXXXXXXXX)
    current_tempfile=$(mktemp tmp.XXXXXXXXXX)
    other_tempfile=$(mktemp tmp.XXXXXXXXXX)

    delete_tempfiles() {
        rm -f "$ancestor_tempfile" "$current_tempfile" "$other_tempfile"
    }
    trap delete_tempfiles EXIT

    ansible-vault decrypt --output "$ancestor_tempfile" "$ancestor_version"
    ansible-vault decrypt --output "$current_tempfile" "$current_version"
    ansible-vault decrypt --output "$other_tempfile" "$other_version"

    git merge-file "$current_tempfile" "$ancestor_tempfile" "$other_tempfile"

    ansible-vault encrypt --output "$current_version" "$current_tempfile"
    ^D
    $ chmod +x ansible-vault-merge

Let's commit our new merge driver to our repository.

    $ git add .gitattributes ansible-vault-merge
    $ git commit -m "Make Git merge Ansible vaults"
    [master 16d6bc5] Make Git merge Ansible vaults
     2 files changed, 27 insertions(+), 1 deletion(-)
     create mode 100755 ansible-vault-merge

Then we'll pick one of our feature branches and rebase it so that it contains
our new merge driver.

    $ git checkout b
    Switched to branch 'b'
    $ git rebase master
    First, rewinding head to replay your work on top of it...
    Applying: Add a last line

Now let's try that merge again.

    $ git merge --no-edit a
    Decryption successful
    Decryption successful
    Decryption successful
    Encryption successful
    Auto-merging secrets.yml
    Merge made by the 'recursive' strategy.
     secrets.yml | 11 ++++++-----
     1 file changed, 6 insertions(+), 5 deletions(-)
    $ ansible-vault view secrets.yml
    ---
    first: line
    top: secret
    last: line

Great! This also works for rebases.

    $ git reset --hard HEAD@{1}
    HEAD is now at d136367
    $ git checkout a
    Switched to branch 'a'
    $ git rebase b
    First, rewinding head to replay your work on top of it...
    Applying: Add a first line
    Using index info to reconstruct a base tree...
    M   secrets.yml
    Falling back to patching base and 3-way merge...
    Decryption successful
    Decryption successful
    Decryption successful
    Encryption successful
    Auto-merging secrets.yml
    $ ansible-vault view secrets.yml
    ---
    first: line
    top: secret
    last: line

Enjoy!

[git-diff-textconv]: https://git-scm.com/docs/gitattributes#_performing_text_diffs_of_binary_files
[git-merge-driver]: https://git-scm.com/docs/gitattributes#_defining_a_custom_merge_driver
