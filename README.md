# jitic

*jitic* - the **JI**RA **Ti**cket **C**hecker - checks the existence of one or more ticket in the [JIRA](https://www.atlassian.com/software/jira) issue tracker. If the tickets exists we will shutdown with exit code 0. Otherwise with 1.

## Usage

```
Usage of ./jitic:
  -pass="": JIRA Password.
  -stdin=false: If set to true you can stream "-tickets" to stdin instead of an argument. If set "-tickets" will be ignored.
  -tickets="": Message to retrieve the tickets from.
  -url="": JIRA instance URL (format: scheme://[username[:password]@]host[:port]/).
  -user="": JIRA Username.
  -verbose=false: If activated more information will be written to stdout .
  -version=false: Outputs the version number and exits.
```

### Examples

Check if ticket *WEB-22861* exists in *https://jira.example.org/* from parameter.

```bash
./jitic -url="https://jira.example.org/" -user="JIRA-API" -pass="SECRET-PASSWORD" -tickets="WEB-22861 remove authentication prod build for now"
```

Check if ticket *WEB-22861* exists in *https://jira.example.org/* from stdin.

```bash
echo "WEB-22861 remove authentication prod build for now" | ./jitic -url="https://jira.example.org/" -user="JIRA-API" -pass="SECRET-PASSWORD" -stdin
```

## Use cases

### Subversion "pre-commit" hook

See [Implementing Repository Hooks](http://svnbook.red-bean.com/en/1.7/svn.reposadmin.create.html#svn.reposadmin.create.hooks) and [pre-commit](http://svnbook.red-bean.com/en/1.7/svn.ref.reposhooks.pre-commit.html).

How a pre-commit hook can look like:
```sh
#!/bin/sh

REPOS="$1"
TXN="$2"

# Get the commit message
SVNLOOK=/usr/bin/svnlook
COMMIT_MSG=$($SVNLOOK log -t "$TXN" "$REPOS")

JITIC=/usr/bin/jitic
JIRA_URL="https://jira.example.org/"
JIRA_USERNAME="JIRA-API"
JIRA_PASSWORD="SECRET-PASSWORD"

# Exit on all errors.
set -e

# Auth against JIRA and check if the ticket(s) exists
$JITIC -url="$JIRA_URL" -user="$JIRA_USERNAME" -pass="$JIRA_PASSWORD" -tickets="$COMMIT_MSG"

# All checks passed, so allow the commit.
exit 0
```

**Pro-Tip**: Set the variables *JIRA_USERNAME* and *JIRA_PASSWORD* in a seperate file and import this file via [*source*](http://www.tldp.org/HOWTO/Bash-Prompt-HOWTO/x237.html) into the hook.
With this you can store the pre-commit hook itself in git + deploy it with configuration management.

### Git "pre-receive" hook

See [Customizing Git - Git Hooks](https://git-scm.com/book/it/v2/Customizing-Git-Git-Hooks) and [A reasonable git pre-receive-hook](https://gist.github.com/caniszczyk/1327469) and [Can git pre-receive hooks evaulate the incoming commit?](http://stackoverflow.com/questions/22546393/can-git-pre-receive-hooks-evaulate-the-incoming-commit).

How a pre-receive hook can look like:
```sh
#!/bin/sh

GIT=/usr/local/bin/git
JITIC=/usr/bin/jitic
JIRA_URL="https://jira.example.org/"
JIRA_USERNAME="JIRA-API"
JIRA_PASSWORD="SECRET-PASSWORD"

FAIL=""

validate_ref()
{
	# Arguments
	oldrev=$($GIT rev-parse $1)
	newrev=$($GIT rev-parse $2)
	refname="$3"

	# $oldrev / $newrev are commit hashes (sha1) of git
	# $refname is the full name of branch (refs/heads/*) or tag (refs/tags/*)
	# $oldrev could be 0s which means creating $refname
	# $newrev could be 0s which means deleting $refname

	case "$refname" in
		refs/heads/*)
			# We currently only care about updating branches.
			# If you want to take care for deleting branched check this:
			#   if [ 0 -ne $(expr "$newrev" : "0*$") ]; then
			#       # Your code here
			#   fi

			# Pushing a new branch
			if [ 0 -ne $(expr "$oldrev" : "0*$") ]; then
				COMMITS_TO_CHECK=$($GIT rev-list $newrev --not --branches=*)

			# Updating an existing branch
			else
				COMMITS_TO_CHECK=$($GIT rev-list $oldrev..$newrev)
			fi

			# If we push an new, but empty branch we can exit early.
			# In this case there are no commits to check.
			if [ -z "$COMMITS_TO_CHECK" ]; then
				return
			fi

			# Get all commits, loop over and check if there are valid JIRA tickets
			while read REVISION ; do
				COMMIT_MESSAGE=$($GIT log --pretty=format:"%B" -n 1 $REVISION)

				$JITIC -url="$JIRA_URL" -user="$JIRA_USERNAME" -pass="$JIRA_PASSWORD" -tickets="$COMMIT_MESSAGE"
				if [ $? != 0 ]; then
					FAIL=1
					echo >&2 "... in revision $REVISION"
				fi
			done <<< "$COMMITS_TO_CHECK"

			return
			;;
		refs/tags/*)
			# Support for tags (new / delete) needs to be done.
			# Things we need to check:
			#   * Get all commits from the new tag that are NOT checked yet
			#     (checked means by jitic). Something like commits which are
			#     not pushed yet, but the tag was pushed.
			# I think we don`t need to care about deleted branches yet.
			;;
		*)
			FAIL=1
			echo >&2 ""
			echo >&2 "*** pre-receive hook does not understand ref $refname in this repository. ***"
			echo >&2 "*** Contact the repository administrator. ***"
			echo >&2 ""
			;;
	esac
}

while read OLD_REVISION NEW_REVISION REFNAME
do
	validate_ref $OLD_REVISION $NEW_REVISION $REFNAME
done

if [ -n "$FAIL" ]; then
	exit $FAIL
fi
```

**Pro-Tip**: Set the variables *JIRA_USERNAME* and *JIRA_PASSWORD* in a seperate file and import this file via [*source*](http://www.tldp.org/HOWTO/Bash-Prompt-HOWTO/x237.html) into the hook.
With this you can store the pre-commit hook itself in git + deploy it with configuration management.

## License

This project is released under the terms of the [MIT license](http://en.wikipedia.org/wiki/MIT_License).
