# Miscellaneous Implementation Notes

## Native modules

Having downloaded and built the node addon examples from:

https://github.com/nodejs/node-addon-examples

specifically: 1_hello_world/nan

I managed to rebuild manually for Electron by doing:

    $ HOME=~/.electron-gyp node-gyp rebuild --target=1.4.12 --arch=x64 --dist-url=https://atom.io/download/electron

Instructions cribbed from electron-rebuild page.
Note: running electron-rebuild isn't sufficient.

------
Attempts to build (forked) sqlite3 module from source didn't succeed.
What I tried:

1. Running the above node-gyp command direct from command line in the same directory.  Failed with:
 gyp: Undefined variable module_name in binding.gyp while trying to load binding.gyp

Digging in a bit:

    - There is a section "binary" in "package.json" that defines these variables.
    - But: It appears that these are only understood by node-pre-gyp, not node-gyp.

Unfortunately, just replacing node-gyp with node-pre-gyp didn't work.
-------


Arrrrrg.  And now....where did I get the 1.4.12 version from?
Looks like it's the version number in Tad's package.json.
But running 'electron --version' for globally installed electron yields
'v6.5.0'
....and, indeed, running:
$  ./node_modules/.bin/electron --version
v1.4.7

Huh.
Oh, and it turns out Tad had version 1.4.7 of electron installed even
though package.json reported 1.4.12.  GRRRRR.

Let's trying bringing everything up to latest (1.4.13):

first:  Update Tad to 1.4.13 of electron:

....unfortunately that completely chokes; electron-rebuild no longer sufficient to build sqlite3.

10 Jan. 17:

Some mistakes / misunderstandings / bugs in the above:

   - electron reporting its version as 'v6.5.0' was due to running
   electron with ELECTRON_RUN_AS_NODE=true. Let's not do that!
   - Due to a yarn bug, Tad didn't have the version of electron I thought it did.  workaround: Remove node_modules and yarn.lock before running
   yarn (or just go back to npm client).
   yarn bug: https://github.com/yarnpkg/yarn/issues/2423
   - A bug in the latest electron-rebuild won't correctly build sqlite3.
   Workaround: reverting to electron-rebuild 1.4.0. Issue: https://github.com/electron/electron-rebuild/issues/137

----
So, when running electron-rebuild 1.4.0 with -l ('log'), apparently it is just doing:
    node-pre-gyp install --fallback-to-build

but I'm guessing it sets some env vars or something to force pre-gyp to build specifically for correct ABI for electron.

Since I believe --fallback-to-build is falling back to doing a native build, I tried going back to doing:

$ HOME=~/.electron-gyp node-gyp rebuild --target=1.4.14 --arch=x64 --dist-url=https://atom.io/download/electron

but this fails with:
gyp: Undefined variable module_name in binding.gyp while trying to load binding.gyp

Note that this variable (module_name) is referenced in binding.gyp, but defined in the "binary" section of package.json like so:

"binary": {
  "module_name": "node_sqlite3",
  "module_path": "./lib/binding/{node_abi}-{platform}-{arch}",
  "host": "https://mapbox-node-binary.s3.amazonaws.com",
  "remote_path": "./{name}/v{version}/{toolset}/",
  "package_name": "{node_abi}-{platform}-{arch}.tar.gz"
},

So our options seem to be:

1. Just create another electron project for development / testing that will use a file path to install my node-sqlite3 fork, and run electron-rebuild in that dev project as usual.  Just reinstall in the parent/dev project whenever I want to test changes to my fork.

2. Figure out how to craft the env in the same way electron-builder does to run node-pre-gyp directly so it picks up the right info from package.json, etc. but builds for right node_abi for electron.

3. Figure out how node-pre-gyp runs node-gyp (if it does so?) and then set things up to run that directly, adding in the args and HOME to get correct ABI and electron headers.

Let's explore (2) first, then (3), then (1).

.....

Most expedient is actually probably (1), especially since we can just try building with node-pre-gyp, testing with node, and then try building
in the dev project with electron.
First let's set up a test project and make sure this cunning plan will work....

OK, that worked (yay!), and now have basic text-only import working.
Let's see about using RegEx's for column types...:

------
1/19/17:
OK, at this point we've got a node-sqlite3 and node-sqlite3-noodle project.
In the latter we can do:
$ npm install --build-from-source --sqlite=/usr/local/opt/sqlite ../node-sqlite3
for a (relatively) quick build
It seems this does a:
    node-pre-gyp install --fallback-to-build
...just like electron-rebuild

Problem we are now encountering:
We can do the above relative path npm install of my fork of node-sqlite3, BUT when we then do an electron-rebuild, we get an error that it's unable to find the <regex> header.
So...let's try and manully construct a call to node-pre-gyp that will work.

From looking at README.md for webkit, looks like we might be able to get away with passing --target and --target_arch directly to npm install, something like:

$ HOME=~/.electron-gyp npm install ../js/node-sqlite3 --build-from-source --target_arch=x64 --target=1.4.14

...not even close. *sigh*.

Hypothesis:  Maybe there's some difference in common include files in standard node.js vs
electron...looking at this SO answer:
http://stackoverflow.com/a/31112922/3272482

maybe I can look ~/.node_gyp/x.x.x/common.gypi vs electron's ~/.electron-gyp?

----
OK, now have a more minimal test for tracking this down that eliminates node-pre-gyp and electron-rebuild:

In /Users/antony/home/src/js/node-addon-examples/1_hello_world/nan
If we run:
$ node-gyp rebuild
which builds against our local node install (apparently 7.1.0), all works OK.
But if we try:
$ HOME=~/.electron-gyp node-gyp rebuild --target=1.4.14 --arch=x64 --dist-url=https://atom.io/download/electron
we fail to compile the standard <regex> header.

Looking at the output of both, the former appears to use:
/Users/antony/.node-gyp/7.1.0/include/node/common.gypi

whereas the latter uses:
/Users/antony/.electron-gyp/.node-gyp/iojs-1.4.14/common.gypi

yep, seems to be the diff at line 391 of /Users/antony/.node-gyp/7.1.0/include/node/common.gypi

========
Back to Tad:

(Test Edit).

TODO before next release:
  X clean up debug printfs in sqlite import C++
  X row count in C++
  X error signalling from C++ addon
  - UI for import errors in a dialog
  X Open window immediately on Tad start
  X Loading indicator during initial import
  - Electron auto-update

======
3/22/17:

To get auto-update working want to package Tad as DMG with electron-builder and follow the auto-update model of ~/src/js/electron-updater-example

---
I've upgraded all deps to use latest webpack, etc.
Note:
Because of dependency on git URL for my fork of sqlite3, need to install
Need to install deps with npm (not yarn):
$ npm install

To build app:
$ npm run build-app

To build release as DMG:
for now:
$ node_modules/.bin/build --mac

...but will need to work out publishing step to push releases to github.
  See README.md and publish.sh in electron-updater-example

=====
3/30/17:
  - REALLY need to use column type to determine what agg fns to allow in UI;
    avg/count/sum just don't make sense for string columns


=====
Let's look at log data for what argv looks like under the different
launch scenarios:
Open Action: double click on app, context menu open, command line:
npm run vs production app
Platform: Mac, Windows

Mac, Production App:
  - Double Click on App: `[ '/Applications/Tad.app/Contents/MacOS/Tad' ]`
  - Context menu from Finder: `[ '/Users/antony/home/src/tad/dist/mac/Tad.app/Contents/MacOS/Tad',
  '-psn_0_3781531' ]` (followed by `open-file` event)
  - command line (shell wrapper, no arguments):
  `[ '/Applications/Tad.app/Contents/MacOS/Tad',
  '--executed-from=/Users/antony/home/src/tad' ]`
  - command line w/ specific file:
  `[ '/Applications/Tad.app/Contents/MacOS/Tad',
  '--executed-from=/Users/antony/home/src/tad',
  '/Users/antony/data/movie_metadata.csv' ]`

======
Notes, reworking csv import to support European format:

- Need to tweak readHeaderRow to become extractHeaderRow from first line returned from byline based readSampleLines
- need to thread delimiter through importData so that prepValue can kill . and map , to . if European format

======
Upgrading npm dependencies, 10/17/18:
