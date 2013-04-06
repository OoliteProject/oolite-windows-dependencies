# Oolite Windows dependencies
This repository contains binary dependencies used to build Oolite for Windows. It is included as a submodule in the main Oolite repository.

The documented build process for Oolite for Windows will pull in these pre-built dependencies with no additional work required. The rest of this file documents how to reproduce the changes made.


## Modifications to external libraries' source code for running Oolite
The various ports of Oolite are using a number of external libraries for graphics, sound, input and event handling. Throughout development, certain changes to the source code of these libraries have been deemed necessary, either to enable Oolite to run in a more efficient and independent manner, or simply to fix issues that occurred as a result of these external libraries themselves. Of these libraries, the ones that have to be rebuilt specifically for Oolite, together with the main reasons/areas changed for this reason are:

1. gnustep-base v1.20.1 – bug in `NSInteger` definition, change to dictionary format of `NSUserDefaults`, fix for integer truncation of integers to 32-bit when parsing XML plists (bug #32495 on the gnustep.org bug tracker) and changes to facilitate native Objective-C exception support.
2. SDL v1.2.13 – window resizing issues, backported fix for setting gamma crash.
3. SpiderMonkey v1.85 – certain JS macro definitions required by Oolite not guaranteed or non-existent in standard library.
4. eSpeak v1.43.03 – Special build of the Windows speech synthesis libespeak.dll to enable asynchronous playback. Also, defaults the eSpeak data directory to Resources/espeak-data.
5. SDL_mixer v1.2.7 – Bug fix for static heard over Ogg Vorbis music when music loops.

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

The GNUstep objc-1.dll runtime has been rebuilt with native Objective-C exception support. To do this on Windows, the patch which provides the `void (*_objc_unexpected_exception) (id exception)` callback hook to the runtime is required for versions of gcc older than 4.4.0. The patch can be downloaded from http://gcc.gnu.org/bugzilla/attachment.cgi?id=17365. Also, the gcc source header unwind-pe.h must be visible to exception.c in order for the build of libobjc to succeed.

The full source code of GNUstep 1.20.1 is available from
ftp://ftp.gnustep.org/pub/gnustep/core/gnustep-base-1.20.1.tar.gz

### SDL v1.2.13
SDL_resize.c:57: Added the lines
```C
#ifdef __WIN32__
	SDL_VideoSurface->w = w;
	SDL_VideoSurface->h = h;
#endif
```
to enable window resizing without side effects like texture corruption in the Windows port of Oolite. The entire source of the modified SDL library is included in the source distribution of the game under &lt;source code installation folder&gt;/deps/Cross-platform-deps/SDL/SDL-1.2.13.zip

SDL_dibvideo.c:768: Changed from
```C
/* BitBlt() maps colors for us */
video->flags |= SDL_HWPALETTE;
```
to
```C
if ( screen_pal )
{
	/* BitBlt() maps colors for us */
	video->flags |= SDL_HWPALETTE;
}
```
to fix crash occurring when attempting to set the screen gamma value. This fix occurred in a later version of SDL and was backported to the version used with the Windows port of Oolite.

### SpiderMonkey v1.85
Specific build settings for Oolite are required. Library rebuilt with `MOZ_TRACE_JSCALLS` defined in order to enable full JavaScript profiling.

The entire source code of the library can be downloaded from ftp://anonymous@ftp.mozilla.org/pub/firefox/releases/4.0/source/firefox-4.0.source.tar.bz2

### eSpeak v1.43.03
The libespeak.dll has been custom-built for the Windows port of Oolite to enable asynchronous playback and to also default the eSpeak data files folder to Resources/espeak-data. The source files that need to be changed for this to happen can be found inside oolite-windows-dependencies/OOeSpeakWin32DLLBuild. The instructions for building this library, for those interested, are as follows:
* You will need to have OoliteDevelopmentEnvironment - Light Edition installed, downloadable from http://terrastorage.ath.cx/Marmagka/c0a11a8fe8b5124823a91ff1d90065ff/OoliteDevelopmentEnvironment_LE_20110212.zip. Instructions for setting it up can be found at http://www.aegidian.org/bb/viewtopic.php?t=5943.
*   Download espeak-1.43.03-source.zip from http://espeak.sourceforge.net/download.html
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
music.c: Commented out lines 334 and 335 of `music_mixer` function.
