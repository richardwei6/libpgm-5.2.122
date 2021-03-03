
# libpgm-5.2.122
Working &amp; updated OpenPGM source code in 2021. 

I created this repo due to the relic that is libpgm. It is based off the source code that you can download from [here](https://code.google.com/archive/p/openpgm/downloads).

It was last updated all the way back in 2012 and I feel like it's time for a bit of a refresher for people like me who want to use libpgm but don't know how to deal with compiling libraries and stuff by ourselves.

Before I get started, I want to note that this source code was able to successfully compile using CMAKE GUI, Windows 10 Pro (Build 19042), and Visual Studio 2019.

# Prerequisites

Before we can actually build stuff, we need to download patch from [here](http://gnuwin32.sourceforge.net/packages/patch.htm). 

What does patch do? Well, from their website description, "`patch` takes a patch file containing a difference listing produced by diff and applies those differences to one or more original files, producing patched versions."

All you need to do is install patch by following the installer. Make sure to note where you install patch however. The default location should be `C:\Program Files (x86)\GnuWin32` but this obviously differs from what you might choose. From now on, I'll call this location `PATCH_PREFIX` for simplicity.

Next, we need to make sure that we have both Python and Perl installed. We need Python for the .py script and Perl for something in the source code. 

Make sure to restart your computer after installing Perl so that the install location shows up in the enviroment variables.

# The Build

Now, for any cmake library, we first need to create a project that we can compile. For me, I'll be using CMAKE GUI.

 1. Download the source code folder in this repository.
 2. Open CMAKE GUI and navigate the source code to where you downloaded the previous folder
 3. For "where to build binaries", create any random folder and select it in cmake
 4. Next, press configure.
 5. Select the correct options according to your system and press Finish.
 6. Now, you should see 5 configurable lines.
 7. For the configureable variable `PATCH_EXECUTABLE`, we need to set it as `PATCH_PREFIX/bin/patch.exe`.
 8. Press generate.
 9. Now, we can open the project using Visual Studio and compile.
 10. It should compile sucessfully and in your output folder that you choose, you can find the .lib in `/lib`
