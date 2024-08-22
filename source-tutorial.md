# Git LFS Tutorial

First things first, install git 1.8.2 or newer and then install git-lfs (See [here](Installation))

## Create a new repo ##

    git init .
    echo Hello World > README.md
    git add README.md
    git commit -m "Initial commit"

Let's create some more files

    ls > foo.txt
    ls > bar.txt

And add them to git

    git add foo.txt bar.txt

You should already have git-lfs [installed](Installation). To ensure that git-lfs is setup correctly in your git configuration files use the `git lfs install` command:

    git lfs install

Now let's add some large files to be tracked by git-lfs:

    dd if=/dev/urandom of=cat.bin bs=1048576 count=1
    dd if=/dev/urandom of=dog.bin bs=1048576 count=1

Now we are going to ensure that git-lfs is tracking the large `*.bin` files we just created. Tracking means that in subsequent commits, these files will now be LFS files.

*Note: This does NOT mean that versions of the files in previous commits will be converted. That involves a process commonly known as "rewriting history" and is described in the [migration section](#migrating-existing-repository-data-to-lfs).*

We do this by setting a `track` pattern, using the `git lfs track` command.

    git lfs track '*.bin'

This tells git-lfs to track all files matching the `*.bin` pattern. The quotes around the `*.bin` are there to prevent the shell from expanding `*.bin`, else you would end up only tracking `cat.bin dog.bin`, and no other `bin` in the future.

To see a list of all patterns currently being tracked by git-lfs, run `git lfs track` with no arguments

```
Listing tracked paths
    *.bin (.gitattributes)
```

To see the list of files being tracked by git-lfs, run `git lfs ls-files`. You will see that this list is currently empty. This is because technically the file isn't an lfs object until after you commit it.

Next, you need to add `.gitattributes` to your git repository. `git lfs track` stores the tracked files patterns in `.gitattributes`. This way when the repo is cloned, the tracked file patterns are preserved.

    git add .gitattributes

We can also add the `*.bin` files now that they are tracked via git-lfs.

    git add "*.bin"

Now `git status` should look like this.

```
On branch master
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

        new file:   .gitattributes
        new file:   bar.txt
        new file:   cat.bin
        new file:   dog.bin
        new file:   foo.txt
```

Finally, commit the new files

    git commit -m "Add files"

Now, when you run `git lfs ls-files`, you will see the list of tracked files

```
f05131d24d * cat.bin
7db207c488 * dog.bin
```

## LFS URL ##

A default Git LFS endpoint URL is determined when you clone a Git repository or add a Git remote.  The endpoint identifies the server URL to which the Git LFS objects are sent.

To see the Git LFS endpoint URL, run `git lfs env`:

```
git-lfs/1.1.1 (GitHub; windows amd64; go 1.5.3; git 7de0397)
git version 2.6.4.windows.1

Endpoint=https://example.com/path/myrepo.git/info/lfs (auth=none)
Endpoint (upstream)=example.com/path/myrepo.git/info/lfs (auth=none)
LocalWorkingDir=C:\src\myrepo
LocalGitDir=C:\src\myrepo\.git
LocalGitStorageDir=C:\src\myrepo\.git
LocalMediaDir=C:\src\myrepo\.git\lfs\objects
TempDir=C:\src\myrepo\.git\lfs\tmp
ConcurrentTransfers=3
BatchTransfer=true
git config filter.lfs.smudge = "git-lfs smudge %f"
git config filter.lfs.clean = "git-lfs clean %f"
```

You may sometimes need to override the default Git LFS endpoint URL; for instance, if you are using a Git LFS server like [lfs-test-server](https://github.com/github/lfs-test-server) rather than the one offered by your Git hosting service. Overriding the default endpoint URL can be done using the `lfs.url` Git configuration option, e.g.:

```
$ git config lfs.url "https://my_other_server.example.com/foo/bar/info/lfs"
```

However, this modifies the `.git/config` file, which is a local Git file, and which therefore cannot be added to your Git repository. To get around this and allow the Git LFS endpoint URL to be saved in the Git repository's contents, so it will be automatically defined for anyone who clones the repository, Git LFS also supports the use of the custom `.lfsconfig` file.

```
$ git config -f .lfsconfig lfs.url "https://my_other_server.example.com/foo/bar/info/lfs"
$ git add .lfsconfig
```

Now, `git lfs env` shows:

```
git-lfs/1.1.1 (GitHub; windows amd64; go 1.5.3; git 7de0397)
git version 2.6.4.windows.1

Endpoint=https://my_other_server.example.com/foo/bar/info/lfs (auth=none)
...
```

*Note: `http://` and `https://` URL endpoints are supported by all versions of the Git LFS client. As of v2.9.0, there is also support for `file:///` URLs; note that the colon character must be escaped as `%3a`. Local file paths are supported as of v2.10.0. A Git LFS transport protocol using only SSH is available in the Git LFS client as of v3.0.0, allowing full `ssh://` URL support, but only if the Git LFS server also provides this support. Plain HTTP should never be used over the public Internet, as your credentials will not be securely encrypted; using HTTPS is always preferred.*

*Note: The [[Implementations]] page lists a large number of available open-source and commercial Git LFS server implementations, some of which now support the full SSH transport option for Git LFS.*

## Credential caching/storage ##

Since most commercial Git LFS storage services only support the HTTPS version of the Git LFS transfer protocol at this time, Git LFS will need to authenticate over HTTPS when pushing files, even if you are using the SSH protocol for regular Git operations. Without a credential helper, you will be asked to enter your username and password for _every_ connection, which is pretty unusable. To get around this, Git credential helpers will help by handling passwords for you.

Various options are available for different platforms.  One cross-platform option is the [Git Credential Manager](https://github.com/git-ecosystem/git-credential-manager), which is available for Windows, macOS, and Linux.  Other options include the `osxkeychain` helper provided with Git on macOS, and the `cache` [helper](https://git-scm.com/docs/git-credential-cache) for Linux, and additional options are [listed](https://git-scm.com/doc/credential-helpers) in the Git documentation.

## Adding Git LFS to a pre-existing repo  ##

Let's create pretend repo to understand how converting to git-lfs works

```
git init .
touch README.md
git add README.md
git commit -m "initial"
git tag one
echo hi > plain.txt
ls > foo.bin
git add plain.txt foo.bin
git commit -m "Add some files"
git tag two
echo bye > plain.txt
ls > bar.bin
ls > foo.bin
git add plain.txt foo.bin bar.bin
git commit -m "Update and add another file"
git tag three
echo bye >> plain.txt
ls > foo.bin
git add plain.txt foo.bin
git commit -m "Update some more"
git tag four
```

Now lets decide we want `*.bin` files to be turned into lfs objects.

```
git lfs track '*.bin'
git add .gitattributes
git commit -m "Track .bin files"
git tag not_working
```

Just tracking files does NOT convert them to lfs. They are part of history already, the only way to convert old files is to rewrite history (see the section about [migrating existing data](#migrating-existing-repository-data-to-lfs)).

Next we can try converting the latest version of history only to LFS. Since `*.bin` are currently tracked, we just need to add the files back after removing them. We will remove it from git without deleting the file (aka remove cache), and then add it back

```
git rm --cached *.bin
git add *.bin
git commit -m "Convert last commit to LFS"
```

Now, `git lfs ls-files` shows:

```
4665a5ea42 * bar.bin
4665a5ea42 * foo.bin
```

We can also see that the `foo.bin` is an lfs file by `git show HEAD:foo.bin`

```
version https://git-lfs.github.com/spec/v1
oid sha256:4665a5ea423c2713d436b5ee50593a9640e0018c1550b5a0002f74190d6caea8
size 36
```

But when we look at the previous histories, we see that they are still not LFS tracked. `git show not_working:foo.bin`

```
bar.bin
foo.bin
plain.txt
README.md
```

It's important to understand that tracking an LFS file does not remove it from a previous version of history. If you create a repo without LFS before, and put hundreds of MB in there, after all these steps, the old version still has all the large files in git, not in LFS. There is a discussion on how to rewrite history [here](https://github.com/github/git-lfs/issues/326), but in general the best tool depends on the use case. See the next section for an introduction to some of the available tools.

## Migrating existing repository data to LFS ##

Sometimes files end up committed within your repository when they should have been committed with LFS. This can be accidental, or can come up when converting an existing Git repository to use Git LFS. To make this possible, Git LFS ships with a `git lfs migrate` command that allows for such migrations:

### Using `git lfs migrate` ###

* Install Git LFS v2.2.1 (or later)
* Rewrite e.g. all `*.mp4` video files on the current branch that are not present on a remote:

```
git lfs migrate import --include="*.mp4"
```

* Alternatively, rewrite all `*.mp4` video files on a given branch(es) regardless of whether they are present on a remote (may require a force-push):

```
git lfs migrate import --include="*.mp4" --include-ref=refs/heads/master --include-ref=refs/heads/my-feature
```

* Push the converted repository as a new repository:

```
git push --force
```

### Cleaning up the .git directory after migrating

The above successfully converts pre-existing git objects to lfs objects. However, the regular objects still persist in the .git directory. These will be cleaned up eventually by git, but to clean them up right away, run:

```
git reflog expire --expire-unreachable=now --all
git gc --prune=now
```

## Pulling and cloning ##

Once git lfs is installed, to clone an LFS repo, just run a normal `git clone` command

    git clone https://github.com/username/my_lfs_repo.git destination_dir

You can also clone using the git protocol, LFS assets will still be pulled down via http, which may incur a password prompts depending on your LFS server, since preshared sshs key will not be used for authentication on LFS assets.

    git clone git@github.com:username/my_lfs_repo.git destination_dir

With a reasonably modern Git, there are no longer any performance problems with using a standard `git clone` command.

TODO:

* Push
* Pushing missing files `git lfs push origin master --all`
* Cloning
* Pulling as clone, pulling after clone, etc...
* Pulling missed files `git checkout .`, `git lfs ls-files` shows `-` for missing pulled files(smudged), and `*` for pulled files(cleaned)
* Pulling all LFS objects (to conveniently work offline) `git lfs fetch --all`, there is also a `--recent` flag to only fetch the last few days. See `git lfs fetch --help` for more details

## LFS pointer files (Advanced) ##

Git LFS stores a pointer file in the git repo in lieu of the real large file. The pointer is swapped out for the real file at checkout (using `smudge` and `clean`).  The `smudge` and `clean` filters are part of core Git and are designed to allow changing a file on checkout (`smudge`) and on commit (`clean`).  Git LFS uses these techniques to replace the pointer files with the actual large files that are in use.

Normally, you would never see it, but for the curious, you can view the pointer file by `git show HEAD:{filename}`, for example:

`cat cat.bin`

    bar.txt
    cat.bin
    foo.txt
    README.md

`git show HEAD:cat.bin`

    version https://git-lfs.github.com/spec/v1
    oid sha256:f05131d24da96daa6a6712c5b9d368c81eeaea5dc7d0b6c7bec7d03ccf021b4a
    size 34

You can also create any pointer you want, `cat dog.bin | git lfs clean`

    version https://git-lfs.github.com/spec/v1
    oid sha256:7db207c488a5957f0b88aec97489a60e73f2b8d310586c2502f3388af7f76091
    size 42

Running `git lfs clean` like this, also creates a new lfs object for every clean. Not sure why you'd want to. For example: `echo hi | git lfs clean`:

    version https://git-lfs.github.com/spec/v1
    oid sha256:98ea6e4f216f2fb4b69fff9b3a44842c38686ca685f3f55dc48c5d3fb1107be4
    size 3

Now, `ls .git/lfs/objects/98/ea` shows:

    98ea6e4f216f2fb4b69fff9b3a44842c38686ca685f3f55dc48c5d3fb1107be4

You can also generate a pointer using `git lfs pointer`, but does not create lfs objects.

## Untrack and Remove files from LFS ##

You can also untrack all files of a certain type from lfs and remove it from the cache:

```
git lfs untrack "*file-type"
git rm --cached "*file-type"
```
If you would like to add these files back to regular git tracking and commit them, you can do the following:

```
git add "*file-type"
git commit -m "restore "*file-type" to git from lfs"
```

note: "\*file-type" is formatted as "\*.mp4"
