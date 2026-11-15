# sid-management

This repository provides tooling and cached information that could be used by designated experts of the [yang-sid registry](https://www.iana.org/assignments/yang-sid) to interact with IANA.

Warning: this currently uses whatever version of [pyang](https://github.com/mbj4668/pyang) you have installed.
pyang 2.7.1 has a known problem with handling SIDs that include choices and cases.

SPDX-License-Identifier: MIT

***

# rake targets:

```
rake manual_rsync -- pull in manually selected yang modules
rake rfcmod_rsync -- get all rfcmod yang modules from IANA

rake build_sids -- build the SID files and stats.csv

rake rsync = rake manual_rsync rfcmod_rsync

rake default = rake rsync build_sids

rake clean -- remove sid files from build directory
```
