==============================================================
 LINUX INODE & FILE CONCEPTS — REFERENCE NOTES
==============================================================

--------------------------------------------------------------
1. WHAT AN INODE IS
--------------------------------------------------------------
An inode (index node) is the on-disk structure storing a file's
METADATA and BLOCK POINTERS — but NOT its name and NOT its
contents directly.

An inode holds:
  - File type (regular, dir, symlink, socket, block/char dev, FIFO)
  - Permissions (mode bits) + owner UID / group GID
  - Size
  - Timestamps: atime / mtime / ctime
      (no POSIX creation time; ext4/xfs have crtime but not POSIX)
  - Link count (how many hard links point to it)
  - Pointers to the actual data blocks

An inode does NOT hold:
  - The filename -> that lives in the DIRECTORY, which maps
    name -> inode number.

  directory entry            inode (metadata)         data blocks
  +--------------+          +------------------+      +----------+
  | "app.log"->42|--------> | inode 42         |----> | content  |
  +--------------+          | mode,uid,size    |      +----------+
                            | links=1,blocks[] |
                            +------------------+

--------------------------------------------------------------
2. PRODUCTION CONSEQUENCES
--------------------------------------------------------------
1) Hard links share one inode. Same inode number, link count > 1.
   Deleting one name decrements the count; data frees at count 0.
     ls -li file        # first column = inode number
     stat file          # full metadata + link count

2) rm does not delete data - it unlink()s a NAME. If a process
   still holds the file open, data survives (link count 0 but
   open FD > 0). Deleting a huge log won't free disk until the
   holder truncates/restarts.
     lsof +L1                       # files w/ link count 0 held open
     lsof -p <pid> | grep deleted
     : > /path/to/logfile           # truncate in place instead of rm

3) "No space left on device" while df shows free space =
   INODE EXHAUSTION (millions of tiny files), not byte exhaustion.
     df -i          # inode usage per filesystem
   ext4: inode count fixed at mkfs (-N / -i bytes-per-inode).
   xfs : allocates inodes dynamically, rarely hits this.

4) mv on the SAME filesystem is atomic & cheap - just rewrites
   the directory entry (name->inode), no data copy. Across
   filesystems it's a real copy+delete (different inode space).

5) ctime != creation time. It's INODE CHANGE time - updated on
   chmod/chown/rename/link changes, not just content edits.

--------------------------------------------------------------
3. HANDY COMMANDS
--------------------------------------------------------------
  stat -f /mount     # fs-level: total/free inodes, block size
  find /path -xdev -printf '%i\n' | sort -u | wc -l   # distinct inodes
  debugfs -R "stat <42>" /dev/sdaX   # ext4: dump raw inode by number

--------------------------------------------------------------
4. COMMAND EXECUTION PATH (bonus)
--------------------------------------------------------------
  you type -> shell parses -> resolves command -> fork+exec
           -> kernel runs it -> exit code returned

  Resolution order: alias -> function -> builtin -> $PATH executable
     type -a ls      # shows alias/builtin/file, all matches
     command -v ls   # what will actually run
     echo $?         # exit status of last command (0=success)

--------------------------------------------------------------
5. SYMLINK vs HARDLINK — DEEP DIVE
--------------------------------------------------------------
CREATE:
  ln    target linkname     # hard link
  ln -s target linkname     # symbolic (soft) link

CORE DIFFERENCE:
  - HARD LINK  = a second NAME (directory entry) pointing at the
    SAME inode. Both names are equal peers; neither is "the
    original". Data lives until link count hits 0.
  - SYMLINK    = its OWN separate inode of type 'l' whose data
    block just stores a PATH STRING to the target. It's a
    pointer-by-name, resolved at access time.

  hard link:                        symlink:
    name A --+                        name S (inode 99, type l)
             |--> inode 42 --> data       |  data = "path/to/target"
    name B --+                            v
                                       name T --> inode 42 --> data

INSPECT:
  ls -li            # hard links share the inode #, link count > 1
  ls -l             # symlink shown as  S -> target
  readlink -f S     # resolve symlink to final absolute path
  stat S            # type shows "symbolic link"; %h = link count

COMPARISON TABLE:
  Property              Hard link            Symlink
  -------------------   ------------------   --------------------
  Own inode?            No (shares target)   Yes (type 'l')
  Cross filesystem?     NO (same fs only)    YES
  Link to directory?    NO (root/fsck only)  YES
  Survives target rm?   YES (data persists)  NO (becomes dangling)
  Points to?            inode                pathname (string)
  Extra disk for data?  ~none                stores the path
  Broken if target moves? No                 YES (dangling link)

KEY GOTCHAS:
  1) Hard links CANNOT cross filesystems - inode numbers are only
     unique within one fs. Symlinks can (they store a path).
  2) Delete the target of a symlink -> DANGLING link (points to
     nothing). Delete one hard link -> other name & data survive.
  3) Hard links to directories are forbidden (would create loops
     and break fsck). Symlinks to dirs are normal and everywhere.
  4) Symlinks add a resolution step + can point outside a chroot/
     mount -> a security & path-traversal consideration.
  5) Editors that "save" by writing a new file + rename can BREAK
     a hard link (the name now points to a new inode) but FOLLOW
     a symlink (still edits the real target). Behavior varies by
     editor - worth knowing for config files under /etc.
  6) tar/rsync: rsync -H preserves hard links; -l copies symlinks
     as symlinks; -L follows/derefs them into real files.

WHEN TO USE WHICH:
  - Symlink: cross-fs, link to dirs, human-readable indirection,
    versioned "current -> release-2026-07-01" pointers. (Default.)
  - Hard link: dedup identical files on ONE fs without extra space,
    snapshot-style backups (rsync --link-dest), keep data alive
    under multiple names.

--------------------------------------------------------------
6. INODE EXHAUSTION — TROUBLESHOOTING RUNBOOK
--------------------------------------------------------------
SYMPTOM:
  - Writes fail with "No space left on device" (ENOSPC)
  - BUT df -h shows plenty of free bytes
  - App/log errors: cannot create file, mkdir fails, etc.
  => Suspect INODE exhaustion, not block exhaustion.

STEP 1 - CONFIRM IT'S INODES:
  df -i                      # look for IUse% at or near 100%
  df -ih                     # human-ish; compare IFree vs free bytes
  # If IUse% = 100 but df -h has space -> confirmed inode exhaustion.

STEP 2 - IDENTIFY THE OFFENDING FILESYSTEM:
  df -i | awk 'NR==1 || $5+0 > 80'   # list fs over 80% inode use
  # Note the mountpoint (e.g. /var). Work within -xdev to stay on it.

STEP 3 - FIND WHERE THE TINY FILES ARE:
  # Count files per immediate subdir on the affected fs:
  cd /var    # (the full mountpoint)
  for d in */; do
    printf '%8d  %s\n' "$(find "$d" -xdev | wc -l)" "$d"
  done | sort -rn | head

  # Or one-liner to find directories with the most entries:
  find /var -xdev -type d -printf '%h\n' 2>/dev/null \
    | sort | uniq -c | sort -rn | head -20

  # Common culprits:
  #   /var/spool/{mail,postfix,clientmqueue}  (mail backlog)
  #   session/cache dirs (PHP sessions, tmp)
  #   /var/lib/... app caches, orphaned build artifacts
  #   millions of rotated/small logs

STEP 4 - VERIFY BEFORE DELETING:
  find /path/to/dir -xdev -type f | head            # eyeball samples
  find /path/to/dir -xdev -type f | wc -l           # count
  find /path/to/dir -xdev -type f -mtime +30 | wc -l # how many are old
  du -sh /path/to/dir                               # bytes (often tiny)

STEP 5 - REMEDIATE (safely, in batches):
  # Delete old files without blowing up the arg list / IO:
  find /path/to/dir -xdev -type f -mtime +30 -delete
  # or throttled:
  find /path/to/dir -xdev -type f -mtime +30 -print0 \
    | xargs -0 -n 1000 rm -f

  # Mail queue example (verify with mailq first!):
  # postsuper -d ALL          # flush entire postfix queue (destructive)

  # Remember: if a process holds files open, rm won't free inodes
  # until it releases them:  lsof +L1

STEP 6 - CONFIRM RECOVERY:
  df -i                      # IUse% should drop
  # retry the failing write / restart the affected service

ROOT-CAUSE / PREVENTION:
  - Add monitoring on inode usage, not just disk bytes:
      df -i alerting at ~80% IUse% per critical fs
  - Fix the source: log rotation (logrotate), session GC (PHP
    session.gc_*), cron cleanup of tmp/cache/spool dirs.
  - ext4: inode count is FIXED at mkfs. If chronically short,
    reformat with more inodes:
      mkfs.ext4 -N <count> /dev/sdX      # explicit inode count
      mkfs.ext4 -i <bytes-per-inode> ... # smaller ratio = more inodes
    (Check current ratio:  tune2fs -l /dev/sdX | grep -i inode)
  - Consider XFS for workloads with huge numbers of small files -
    it allocates inodes dynamically and rarely exhausts them.

QUICK REFERENCE:
  df -i                                   # is it inodes?
  find <mnt> -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head
  find <dir> -xdev -type f -mtime +N -delete
  lsof +L1                                # deleted-but-open holders
  tune2fs -l /dev/sdX | grep -i inode     # ext4 inode config

--------------------------------------------------------------
7. DIRECTORY & DENTRY INTERNALS
--------------------------------------------------------------
WHAT A DIRECTORY REALLY IS:
  A directory is just a FILE (its own inode, type 'd') whose data
  is a table of entries mapping:  name -> inode number.
  It stores NO metadata about the files - that's in each inode.
  This is why the same inode can have many names (hard links):
  multiple directory entries simply point at the same inode #.

  directory "dir/" (inode 30, type d)
  +------------------------+
  | "."      -> 30         |   self
  | ".."     -> 12         |   parent
  | "app.log"-> 42         |---> inode 42 (metadata) --> data
  | "notes"  -> 57         |
  +------------------------+

  Note: "." and ".." are real entries -> this is why a fresh
  empty dir has a LINK COUNT of 2 (itself + "." ), and each
  subdir adds 1 to the parent's link count (via its "..").
    stat somedir | grep Links   # links = 2 + number of subdirs

ON-DISK LAYOUT (ext4):
  - Classic: linear list of variable-length records (name, inode,
    rec_len, name_len). Lookups are O(n) - slow for huge dirs.
  - htree (dir_index feature): a hashed B-tree index over names
    for fast lookup in large directories. Check/enable:
      tune2fs -l /dev/sdX | grep features   # look for dir_index
  - XFS: B+tree-indexed directories natively.

DENTRY (the KERNEL side - "directory entry cache"):
  A dentry is an IN-MEMORY (VFS) object, NOT an on-disk thing.
  It caches the result of resolving one path component:
      name  <->  inode  (plus parent dentry pointer)
  Purpose: make repeated path lookups fast without re-reading
  directory blocks from disk every time.

  PATH RESOLUTION (walking /home/test_test/RD_Details.txt):
    start at root dentry (/)
      -> look up "home"        (dcache hit? else read dir block)
      -> look up "test_test"
      -> look up "RD_Details.txt"
    Each step: find child dentry -> get its inode -> check perms
    (need +x/search on every directory in the path).

  dentry states: used (live), unused (cached, reclaimable),
                 negative (remembers a name that does NOT exist -
                 speeds up repeated failed lookups).

CACHES INVOLVED (VFS):
  - dcache : name -> dentry (path lookups)
  - icache : inode cache (metadata in memory)
  Inspect / reclaim:
    slabtop                       # watch dentry / inode_cache slabs
    grep -E 'dentry|inode' /proc/slabinfo
    echo 2 > /proc/sys/vm/drop_caches   # drop dentries+inodes
    echo 3 > /proc/sys/vm/drop_caches   # + pagecache (test/debug ONLY)

WHY IT MATTERS (ops angle):
  1) Search permission is +x on a DIRECTORY, not +r. You can
     traverse a dir you can't list, and list one you can't enter.
  2) Renaming/moving within a fs = editing directory entries only
     (name->inode), so mv is atomic & cheap on the same fs.
  3) Huge directories (100k+ entries) are slow on ext4 without
     dir_index; ls sorts + stats every entry (use ls -f, or
     find/getdents, to avoid the sort/stat storm).
  4) Heavy metadata workloads live or die by dcache/icache hit
     rates; dropping caches in prod tanks performance - don't.
  5) A directory's own size (ls -ld) reflects its entry-table
     size; on ext4 it does not shrink when files are removed.

QUICK REFERENCE:
  stat -c '%F %h' dir              # type + link count
  tune2fs -l /dev/sdX | grep -i 'features\|inode'
  slabtop / grep dentry /proc/slabinfo
  ls -f bigdir                     # list without sort/stat

--------------------------------------------------------------
8. HOW SEARCH WORKS (FILE NAME vs CONTENT) — ARCHITECTURE
--------------------------------------------------------------
Two fundamentally different problems:
  (A) find files by NAME/metadata   -> needs DIRECTORY entries (+inode)
  (B) search CONTENT inside files   -> needs DATA BLOCKS (+ regex CPU)

BIG PICTURE (everything bottoms out in the same kernel machinery):

   query --> tool (find/grep/locate)
                 |  syscalls
                 v
            VFS layer  (kernel)
             |        |          |
          dcache   page cache   FS driver (ext4/xfs)
        (name cache)(file bytes)      |
                                      v
                              DISK: inodes + data blocks

--------------------------------------------------------------
1) FIND FILES BY NAME - find  (LIVE tree walk, no index)
--------------------------------------------------------------
  find /home -name "*.log"
    opendir("/home")        # open the directory file
    loop:
      getdents64()          # read a batch of {name -> inode} entries
      for each entry:
        lstat(entry)        # read that entry's INODE (metadata)
        if dir: recurse
        test predicates: name? mtime? size? perms?
        all match -> print
    closedir()

  COST: I/O-bound on METADATA. Every file -> a stat() (inode read).
  Cold cache + millions of files = millions of tiny random reads
  (slow first run, fast second run once dcache/icache are warm).
    -name (string) is cheap; -type/-mtime/-size/-perm force the
    stat. Put cheap predicates first to short-circuit before stat.
    strace -f -e trace=openat,getdents64,newfstatat find /etc -name hosts

--------------------------------------------------------------
2) FIND FILES BY NAME - locate/plocate  (PREBUILT index)
--------------------------------------------------------------
  updatedb (cron/systemd timer) walks the whole FS ONCE -> builds
     /var/lib/plocate/plocate.db  (compressed index of every path)
  locate "*.log"  just READS the DB - no FS walk at query time.

  TRADE-OFF (the lesson):
    find   = always current, slow, O(files) per query
    locate = near-instant, but STALE (only knows last updatedb;
             new files missing, deleted may linger - plocate
             re-stats to hide those)
    updatedb        # refresh index (root)
    locate -S       # plocate DB stats

--------------------------------------------------------------
3) SEARCH CONTENT - grep  (open + READ + match)
--------------------------------------------------------------
  grep "ERROR" app.log
    open("app.log")        # resolve path (dcache) -> inode
    loop:
      read()               # pull DATA BLOCKS via PAGE CACHE
      split buffer on newlines
      run each line through REGEX ENGINE (DFA/NFA)
      match -> print
    close()

  Regex engine = CPU-heavy part. GNU grep is fast because it:
    - Boyer-Moore: skips ahead on fixed strings (jump multiple
      bytes) -> literal search >> complex regex
    - works on raw buffers, avoids copying lines
    - mmap/large reads to move bytes through the page cache

  RECURSIVE content search = BOTH architectures at once:
    grep -r "ERROR" /var/log   # walk tree AND read every file
  Expensive: metadata walk PLUS reading every byte of every file.

--------------------------------------------------------------
4) MODERN FAST TOOLS - ripgrep (rg) / fd
--------------------------------------------------------------
  Beat grep -r / find ARCHITECTURALLY, same kernel primitives:
    - Parallelism: walk+search across cores (classic grep = 1 thread)
    - .gitignore aware: prunes whole subtrees (node_modules/.git)
      -> far fewer stats and reads
    - Fast regex (Rust): finite-automaton + SIMD literal scan
    - Skips binary files (detects NUL early)
    - Memory-mapped, page-cache-backed reads
  => fewer, smarter, PARALLEL syscalls.

--------------------------------------------------------------
MENTAL MODEL
--------------------------------------------------------------
  Search by NAME  -> DIRECTORY entries (+inode if metadata) -> find/locate
  Search CONTENT  -> DATA BLOCKS (+ regex CPU)              -> grep/rg
  Both recursive  -> tree walk + reads, front-loaded by caches -> grep -r/rg
  Index trades freshness for speed                         -> locate/updatedb

  First run slow (cold cache -> disk); second fast (warm dcache/
  icache/page cache). Same caches as Section 7 make repeats quick.

==============================================================
