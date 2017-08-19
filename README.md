# principles

- fuse shows /$mountpoint/$username/$directory
    - username is whatever name comes in when someone creates directory e.g. despiegk
- groups are defined in toml config file
- acl are done by .acl files per directory, counts for subdirs as well
- fuse is mapped onto a backend dir which is $gitrootdir/$gitrepo (acl's used for security)
- fuse keeps each user separate & commits individual after X period

## defs

- overlaydir
   - per user there is an overlay directory where all change are written (keep the /$mountpoint/$username/$directory readonly and all changes go to the overlay dir)
- gitrootdir
   - the root dir under which there are multiple git repo's
   - $gitrootdir/$gitrepo
   - $gitrepo will be on a branch normally e.g. filemanager
   - each user sees the $gitrepo's in line with the .acl
- filedir
   - big files are dumped in filedir under /$ab/$abcdefghi.$extension   (blake hash, first dir for scalability)

## acl config files = Access Control List

name: .acl

format

```
$groupname RWD
$username RWD
```

how:
- cache in redis which directory applies for which .acl file 
- if dir not in redis yet then walk up & find all .acl files which are relevant for this directory, concatenate all .acl files (the deepest path has priority, so if same groupname exists higher, ignore)
- the expiration is 5 min

## config file has

- groups
  - members: usernames see above
  - if member = * then all users
- username -> email link (for git commit)
- commit interval: default 5 min
- redislocation for events

## commit step

- per user:

## write through

- all files are written on copy on write dir per user = **overlaydir**
- when commit interval passed by (DO THIS PER USER !!!!)
  - step 1 find files which are +1MB
     - walk over all user overlaydirs, check if there are new files
     - files +1 MB -> copy to $filedir (or append to source file if file already exists)
     - write $filepath.link metadata file which has $blakehash.$extension inside
  - when files deleted touch $/overlaydir/$filepath.delete file 
     - during commit we see that file was deleted, remove from source metadata file (do not delete the data)
  - for all files < 1MB & also the .link files
     - copy to right $gitdir/$repo
     - for delete remove from right  $gitdir/$repo
  - when all files processed: big & small files
     - commit for known user e.g. despiegk into the right $gitdir/$repo's can be more than 1
     - push the changes for each repo, because we are the only one pusing on this branch it should never fail
     - in a queue in redis put json with (this allows other tools to further process)
        - $path in git
        - $action: N/C/D  (New/Change/Delete)
        - $epoch
        - $username
   - only do this user per user, while doing the checking & copying (which will be very fast): lock change in fuse layer
      - the push should be done async
- result
  - all big files are in $filedir/$ab/$abcde....$ext  (which we will replicate using e.g. syncthing)
  - all small files are in right git repo & committed using right username
  - all link files  are in right git repo & committed using right username (so we don't abuse git)
  - all changes are logged into redis queue
  - security is easy by .acl files in dirs (which are also on git repo's (-:   )
     
     
## filedir

- big files are dumped in filedir under /$filedir/$ab/$abcdefghi.$extension
- blake hash
- first dir for scalability: $ab first 2 chars of blake hash
- 2 files are stored
   - /$filedir/$ab/$abcdefghi.$extension which is the content
   - /$filedir/$ab/$abcdefghi.source which is the links to the sources
- source metadata
   - text file, per source 1 line
   - $giturl | $pathInGitRepo
   - this allows us to see who is referencing this file
   
  
