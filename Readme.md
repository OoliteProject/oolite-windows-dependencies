# Oolite Windows dependencies
This repository contains binary dependencies used to build Oolite for Windows. It is included as a submodule in the main Oolite repository.

Note: As of October 2022, the 32-bit version of the game is no longer supported. 32-bit libraries are frozen and development continues with the 64-bit versions only.


The documented build process for Oolite for Windows will pull in these pre-built dependencies with no additional work required. The rest of this file documents how to reproduce the changes made.


## Modifications to external libraries' source code for running Oolite
The various ports of Oolite are using a number of external libraries for graphics, sound, input and event handling. Throughout development, certain changes to the source code of these libraries have been deemed necessary, either to enable Oolite to run in a more efficient and independent manner, or simply to fix issues that occurred as a result of these external libraries themselves. Of these libraries, the ones that have to be rebuilt specifically for Oolite, together with the main reasons/areas changed for this reason are:

1. gnustep-base v1.20.1 – bug in `NSInteger` definition, change to dictionary format of `NSUserDefaults`, fix for integer truncation of integers to 32-bit when parsing XML plists (bug #32495 on the gnustep.org bug tracker) and changes to facilitate native Objective-C exception support. Also, fix bug in the stack grabbing code at NSDebug.m so that correct stack traces can be obtained also on Windows XP and later.
2. SDL v1.2.13 – window resizing issues, backported fix for setting gamma crash, haptic devices support, mousewheel delta support.
3. SpiderMonkey v1.85 – certain JS macro definitions required by Oolite not guaranteed or non-existent in standard library.
4. eSpeak v1.43.03 – Special build of the Windows speech synthesis libespeak.dll to enable asynchronous playback. Also, defaults the eSpeak data directory to Resources/espeak-data.
5. SDL_mixer v1.2.7 – Bug fix for static heard over Ogg Vorbis music when music loops. [Obsolete since v1.80]

The changes made in the source code of each of these libraries are as follows:

### gnustep-base v1.20.1
GSConfig.h (build generated file): In the section reading
```C
/*
 * Integer type with same size as a pointer
 */
typedef	unsigned int gsuaddr;
typedef	int gssaddr;
typedef	gsuaddr gsaddr;
```

Change
```C
typedef	gsuaddr gsaddr;
```
to
```C
typedef	gssaddr gsaddr;
```
to fix incorrect definition of NSInteger. 

The `NSUserDefaults` system dictionary (.GNUstepDefaults) is written in XML format in GNUstep 1.20.1. This is inconvenient for Oolite, where the more human-friendly OpenStep format for plists is used. The `static BOOL writeDictionary(NSDictionary *dict, NSString *file)` function, at around line 157 of the NSUserDefaults.m file is therefore changed to read as below:
```objc
data = [NSPropertyListSerialization dataFromPropertyList: dict
	       //format: NSPropertyListXMLFormat_v1_0  // no XML format, use OpenStep instead
	       format: NSPropertyListOpenStepFormat
	       errorDescription: &err];
```
	       
When parsing XML plists, integers are truncated to 32-bit, effectively prohibiting the correct parsing of `long long` or `unsigned long long` values. The fix applied in the gnustep-base-1_20.dll distributed with Oolite in order to address this is:
a) Changing line 309 of NSPropertyList.m of the GNUstep base source code distribution from
```objc
ASSIGN(plist, [NSNumber numberWithInt: [value intValue]]);
```
to
```objc
ASSIGN(plist, [NSNumber numberWithLongLong: [value longLongValue]]);
```
and b) Changing line 1103 of the same file from
```objc
result = [[NSNumber alloc] initWithLong: atol(buf)];
```
to
```objc
result = [[NSNumber alloc] initWithLongLong: atoll(buf)];
```

The stack backtrace grabbing code in NSDebug.m works for Linux but fails on Windows. The GSPrivateStackAddresses function has been modified to use the WinAPI CaptureStackBacktrace method, so that it returns a correct stack backtrace also on Windows (64-bit only, 32-bit version not modified because we want to keep backwards compatibility as much as possible). The modified function is as follows:
```objc
NSMutableArray * GSPrivateStackAddresses(void)
{
  NSMutableArray        *stack;
  NSAutoreleasePool     *pool;

#if HAVE_BACKTRACE
  void                  *addresses[1024];
  int                   n = backtrace(addresses, 1024);
  int                   i;

  stack = [NSMutableArray arrayWithCapacity: n];
  pool = [NSAutoreleasePool new];
  for (i = 0; i < n; i++)
    {
      [stack addObject: [NSValue valueWithPointer: addresses[i]]];
    }
#else	// Windows code here
  unsigned              i;
  const int				kMaxCallers = 62;
  void*					callers[kMaxCallers];
  unsigned              n = CaptureStackBackTrace(0, kMaxCallers, callers, NULL);

  stack = [NSMutableArray arrayWithCapacity: n];
  pool = [NSAutoreleasePool new];
  for (i = 0; i < n; i++)
  {
	[stack addObject: [NSValue valueWithPointer: callers[i]]];
  }
#endif	// HAVE_BACKTRACE
  RELEASE(pool);
  return stack;
}
```
The GNUstep objc-1.dll runtime has been rebuilt with native Objective-C exception support. To do this on Windows, the patch which provides the `void (*_objc_unexpected_exception) (id exception)` callback hook to the runtime is required for versions of gcc older than 4.4.0. The patch can be downloaded from http://gcc.gnu.org/bugzilla/attachment.cgi?id=17365. Also, the gcc source header unwind-pe.h must be visible to exception.c in order for the build of libobjc to succeed.

The full source code of GNUstep 1.20.1 is available from
ftp://ftp.gnustep.org/pub/gnustep/core/gnustep-base-1.20.1.tar.gz

### SDL v1.2.13
The files OOSDLdll_x64.patch and OOSDLdll_x86.patch contain all the changes required for re-creating the SDL.dll used by Oolite on Windows. To apply the patches, you will need to have OoliteDevelopmentEnvironment - Light Edition installed, downloadable from https://drive.google.com/file/d/12xoD3sT1D9yDmOBPp0DKJ0HXWD4-dJjd/view?usp=sharing. Instructions for setting it up can be found at http://www.aegidian.org/bb/viewtopic.php?t=5943. Then, download the SDL-1.2.13 source code from https://sourceforge.net/projects/libsdl/files/SDL/1.2.13/SDL-1.2.13.tar.gz/download, extract the folder SDL-1.2.13 to an empty directory, then copy the appropriate patch file to that same directory. Finally, from the Oolite Development Environment console, cd to that folder and execute `patch -s -d SDL-1.2.13 -p1 < OOSDLdll_x64.patch` or `patch -s -d SDL-1.2.13 -p1 < OOSDLdll_x86.patch` and you are ready to cd to the updated SDL-1.2.13 folder and run `make` inside it in order to bvuild SDL.dll.
The changes in brief:
- Enabled window resizing without side effects like texture corruption in the Windows port of Oolite. 
- Fixed crash occurring when attempting to set the screen gamma value. This fix occurred in a later version of SDL and was backported to the version used with the Windows port of Oolite.
- Backported haptic devices support from later SDL version. This allows force feedback joysticks to work in the game.
- Added mousewheel delta support in the Windows port of Oolite for smoother mousewheel end-user experience.
- Fixed Alt key state not being recognized correctly when returning to the app via Alt-Tab. This is a fix backported from SDL 1.2.15.
- Added capability to accept float pixel types fwhen creating OpenGL window.

### SpiderMonkey v1.85
Specific build settings for Oolite are required. Library rebuilt with `MOZ_TRACE_JSCALLS` defined in order to enable full JavaScript profiling.

The entire source code of the library can be downloaded from ftp://anonymous@ftp.mozilla.org/pub/firefox/releases/4.0/source/firefox-4.0.source.tar.bz2

### eSpeak v1.43.03
The libespeak.dll has been custom-built for the Windows port of Oolite to enable asynchronous playback and to also default the eSpeak data files folder to Resources/espeak-data. The source files that need to be changed for this to happen can be found inside oolite-windows-dependencies/OOeSpeakWin32DLLBuild. The instructions for building this library, for those interested, are as follows:
* You will need to have OoliteDevelopmentEnvironment - Light Edition installed, obtainable from the links mentioned earlier in this document.
*   Download espeak-1.43.03-source.zip from http://espeak.sourceforge.net/download.html or directly from its repository ( http://sourceforge.net/projects/espeak/files/espeak/espeak-1.43/espeak-1.43.03-source.zip/download ).
*   Unzip the file to a folder of choice, maintaining the directory structure. We'll refer to this folder as &lt;espeakSourceFolder&gt;.
*  Copy the following files from oolite-windows-dependencies/OOeSpeakWin32DLLBuild to &lt;espeakSourceFolder&gt;/src.
  - Makefile
  - gettimeofday.c
  - speak_lib.cpp
  - speech.h
* Rename the file &lt;espeakSourceFolder&gt;/src/portaudio19.h to portaudio.h.
* Copy the file speak_lib.h from &lt;espeakSourceFolder&gt;/platforms/windows/windows_dll/src to &lt;espeakSourceFolder&gt;/src.
* Start up MinGW/MSYS and change working directory to the &lt;espeakSourceFolder&gt;/src folder.
* Execute `make libespeak.dll` from the command prompt.
* The library should compile and at the end of the build you should have a file named libespeak.dll (the actual binary) and the import library file libespeak.dll.a, for use when you want to link libespeak.dll to your project.

### SDL_mixer v1.2.7
[Obsolete since v1.80]

music.c: Commented out lines 334 and 335 of `music_mixer` function.
