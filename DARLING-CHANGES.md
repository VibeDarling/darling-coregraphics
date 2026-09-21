# Darling-specific changes

Changes here that exist because of Darling's environment rather than because of
a defect in GNUstep Opal. **Do not send these upstream.**

## `Source/OpalText/DarlingGNUstepCompat.h`

OpalText is GNUstep code and uses GNUstep's `ASSIGN`, `RETAIN`, `RELEASE`,
`AUTORELEASE` and `NSWarnMLog`, which GNUstep declares as macros in
`NSObject.h` and `NSDebug.h`. Darling's Foundation is Apple-shaped and does not
have them, so those uses compile as implicit function calls and fail only at
link. This header supplies them with GNUstep's semantics and is force-included
when building OpalText under Darling.

## Build notes

`fontconfig` and FreeType come from Darling's `wrap_elf` shims,
`src/native/libfontconfig.dylib` and `src/native/libFreeType.dylib`, **not**
from `-lfontconfig` / `-lfreetype`. The Darwin-target linker cannot see host
libraries.

Only `Headers/CoreText` may go on the **public** include path. Exposing
`Headers/` wholesale puts Opal's own `CoreGraphics/CGBase.h` ahead of Darling's
and redefines `CFNumberRef` and `CFSetRef` as `void *`, which breaks every
translation unit identically. The framework's own sources may still reach Opal
private headers such as `OPFont.h`; that rule is about consumers.

# Upstreamable

Kept separate so it can be sent to GNUstep later:

- `Headers/CoreText/CTFontCollection.h` gained `CTFontCollectionCopyOptions`,
  `kCTFontCollectionCopyUnique`, `kCTFontCollectionCopyStandardSort` and the
  `CTFontCollectionCopyFontAttribute` declaration. These are real Core Text API
  that Opal's header simply did not declare.
- `Source/OpalText/Fontconfig/OPFontconfigFontCollection.m` has no `#include`s
  at all and cannot compile; it is built by neither Opal's GNUmakefile nor this
  project's CMakeLists, so it is dead rather than broken-in-use.

- `Source/OpalText/OPFontDescriptor.m` used direct `isa` access at lines 111 and
  212. On a modern non-fragile Objective-C runtime, and on ARM64 in particular,
  `isa` is a packed pointer rather than a `Class`, so messaging it messages
  garbage. `-fontDescriptorByAddingAttributes:` therefore segfaults, which takes
  out `CTFontCreateWithName` and with it every normal entry point into font
  resolution. Replaced with `object_getClass(self)`, which is clang's own
  suggested migration; clang flags the original as
  `-Wdeprecated-objc-isa-usage`.

  This is upstream's bug, not a Darling adaptation. It presumably has never run
  on a modern ARM64 runtime, which is why it has survived. Reproducer needs no
  Darling: create an `OPFontconfigFontDescriptor` via `initWithFontAttributes:`
  and call `-fontDescriptorByAddingAttributes:`.
