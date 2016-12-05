---
title: Diffing Ansible vaults in Git
subtitle: "Posted: 2016-12-22 | Author: Victor Koronen"
layout: default
---

I've been working a lot on infrastructure automation type of tasks recently.
This has meant spending a lot of time working with [Ansible][ansible], the tool
our team chose for managing infrastructure as code.

It didn't take long until we ran into the presumably quite common issue of how
to safely track sensitive credentials that need to appear in plain text as parts
of various configuration files. We use Git as our version control system, so
using something like [Git Crypt][git-crypt] would have been a viable option.
Instead we decided to leverage a feature built into Ansible supporting this
use-case, [Ansible's "Vault" feature][ansible-vault]:

> New in Ansible 1.5, "Vault" is a feature of ansible that allows keeping
> sensitive data such as passwords or keys in encrypted files, rather than as
> plaintext in your playbooks or roles. These vault files can then be
> distributed or placed in source control.

For the purpose of demonstration, let's create a sample Git repository tracking
a single Ansible vault file.

    $ echo "opensesame" > key.txt
    $ echo -e "---\ntop: secret" > secrets.yml
    $ echo -e "[defaults]\nvault_password_file=key.txt" > ansible.cfg
    $ ansible-vault encrypt secrets.yml
    Encryption successful
    $ git init .
    $ git add ansible.cfg secrets.yml
    $ git commit -m "Initial commit"
    $ head -1 secrets.yml
    $ANSIBLE_VAULT;1.1;AES256

[ansible]: https://www.ansible.com/
[git-crypt]: https://www.agwa.name/projects/git-crypt/
[ansible-vault]: https://docs.ansible.com/ansible/playbooks_vault.html

Due to the encrypted nature of these files, diffing their raw on-disk contents
is not very helpful. Fortunately, Git can be told how to decrypt their contents
before diffing through a configuration option called
[`textconv`][git-diff-textconv]:

> Sometimes it is desirable to see the diff of a text-converted version of some
> binary files. For example, a word processor document can be converted to an
> ASCII text representation, and the diff of the text shown. /--/
>
> The `textconv` config option is used to define a program for performing such a
> conversion. The program should take a single argument, the name of a file to
> convert, and produce the resulting text on stdout.

Great! But what program can we use for this purpose?
[`ansible-vault view`][ansible-vault-view]!

    $ git config diff.ansible-vault.textconv "ansible-vault view"
    $ echo "secrets.yml diff=ansible-vault" > .gitattributes
    $ git show --oneline secrets.yml
    7f01f90 Initial commit
    diff --git a/secrets.yml b/secrets.yml
    new file mode 100644
    index 0000000..523ff4c
    --- /dev/null
    +++ b/secrets.yml
    @@ -0,0 +1,2 @@
    +---
    +top: secret

Et voilà!

    $ git add .gitattributes
    $ git commit -m "Make Git diff Ansible vaults"

[git-diff-textconv]: https://git-scm.com/docs/gitattributes#_performing_text_diffs_of_binary_files
[ansible-vault-view]: https://docs.ansible.com/ansible/playbooks_vault.html#viewing-encrypted-files
