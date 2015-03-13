This project is an example of how one could implement an execution environment in other Browsers than Chrome so that Chrome Extensions could run in those other browsers.

If you want to try to build it, you need to overlap it with a Chromium source view.

Note that this project is not currently maintained by anybody so it may fail to compile if you use the latest version of Chromium from the tip of the tree because of changes that happened in the code base since the last update of this project. The last update to the project has been made with revision `73311`. So you may need to fix a few things here and there if you want to use it with a more recent version of the Chromium code... Patches are welcome... ;-)

Start with setting up you Chrome view as described here: http://www.chromium.org/developers/how-tos/build-instructions-windows (these are the instructions for building on Windows since CEEE only works on Windows).

Then you need to add `@73311` to the chromium svn path (i.e., `"url"  : "svn://svn.chromium.org/chrome/trunk/src@73311"`) and remove the `safesync_url` entry since you need to use a specified revision.

Then add this entry to your `.gclient` file:
```
  { "name": "src/ceee",
    "url": "https://ceee.googlecode.com/svn/trunk/ceee"
  },
```

You should end-up with something like this:
```
solutions = [
  { "name" : "src",
    "url"  : "svn://svn.chromium.org/chrome/trunk/src@73311",
    "custom_deps" : {
      "src/third_party/WebKit/LayoutTests": None,
     },
  },
  { "name"        : "src/ceee",
    "url"         : "https://ceee.googlecode.com/svn/trunk/ceee",
  }
]
```

Then run the following command to generate the `ceee.sln` file:
```
python.exe build/gyp_chromium ceee/ceee.gyp
```

You can then build the `ceee\ceee_all` project from the `ceee.sln`, or run the [build\_ceee.bat](https://code.google.com/p/ceee/source/browse/trunk/ceee/build_ceee.bat) file.

And finally, you may want to join the mailing list for ceee-changes:
http://groups.google.com/group/ceee-changes?lnk=gcimh