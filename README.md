# Steps writeup

I built the latest update in a current chroot (`$ makechrootpkg -u -r $CHROOT`).
https://wiki.archlinux.org/title/DeveloperWiki:Building_in_a_clean_chroot

I received the same *error* as @semcore (from `parser.rb`) and saw:
https://github.com/WebKit/WebKit/commit/c7d19a492d97f9282a546831beb918e03315f6ef
I applied the changes.

Next *error*:
```
/usr/include/unicode/localpointer.h:561:26: error: parameter declared 'auto'
  561 | template <typename Type, auto closeFunction>
      |                          ^~~~
```
They talk about this here (*not for the same file/line, but the same issue*):
https://bugs.webkit.org/show_bug.cgi?id=171010
and they link to a patch for *that* file/line, this was only meant to be a temporary workaround though.
They intended to switch from c++11 to c++14 and use auto.
I figured it would be easier to use c++14 rather than find and patch all of the affected autos.
An -std in the WTF make flags would be overshadowed by global flags, so I went ahead and set std=c++14 for the entire build.

Next *error*: (call of overloaded ... is ambiguous)
The WTF team was providing implementations for functions provided by c++14 in preparation for the std change, so I commented out the function.
In their stdlibextras.h, I needed to comment out 2 other functions.

Next *error*: (auto is not permitted in this context)
It looks like this is supported in c++17, so I changed -std to c++17.
Also to reduce prints: -Wno for implicit fallthrough, a c++20 warning, and deprecated warnings.
I removed the D_FORTIFY_SOURCE, as this is added to FLAGS during configure by default and was causing "already set" message spam.

Next *error*: DerivedSources/webkit/webkitmarshal.cpp.
This file has `extern "C" {` on the first line, and includes a header which a series of other includes led to:
```
/usr/include/c++/14.2.1/type_traits:69:3: error: template with C linkage
   69 |   template<typename _Tp>
```
webkitmarshal.cpp is created in ./webkitgtk-2.4.11/Source/WebKit/gtk/GNUmakefile.am.
I removed the echo which puts `extern "C" {` at the top, and added an inplace sed to put `extern "C" {` on the 3rd line below the include statement.

Next *error* is what @bartus posted, xml pointers changed const-ness:
https://bugs.webkit.org/show_bug.cgi?id=265128
I added const to the type.

Next *error*:
```
DerivedSources/WebCore/CSSGrammar.cpp:160:10: fatal error: CSSGrammar.hpp: No such file or directory
  160 | #include "CSSGrammar.hpp"
```
I found this commit which addressed an issue relating to Bison 3.7+:
https://github.com/qtwebkit/qtwebkit/commit/d92b11fea65364fefa700249bd3340e0cd4c5b31
I applied the changes.
