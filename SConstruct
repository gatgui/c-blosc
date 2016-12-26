import re
import sys
import glob
import excons
import excons.tools.threads as threads

env = excons.MakeBaseEnv()

cfg = {
   "lz4": (excons.GetArgument("no-lz4", 0, int) == 0),
   "snappy": (excons.GetArgument("no-snappy", 0, int) == 0),
   "zlib": (excons.GetArgument("no-zlib", 0, int) == 0),
   "zstd": (excons.GetArgument("no-zstd", 0, int) == 0),
   "static": (excons.GetArgument("static", 0, int) != 0),
   "sse2": (excons.GetArgument("with-sse2", 1, int) != 0),
   "avx2": (excons.GetArgument("with-avx2", 0, int) != 0)
}

def GenerateConfig(target, source, env):
   e = re.compile(r"^#cmakedefine\s+([^\s]+)\s+(@[^@]+@)")
   with open(str(source[0]), "r") as hin:
      with open(str(target[0]), "w") as hout:
         for line in hin.readlines():
            m = e.match(line)
            if m:
               if m.group(1).startswith("HAVE_"):
                  enable = cfg.get(m.group(1)[5:].lower(), True)
                  line = line.replace(m.group(2), "1" if enable else "0")
               elif m.group(1) == "BLOSC_DLL_EXPORT":
                  line = line.replace(m.group(2), "0" if cfg["static"] else "1")
               line = line.replace("#cmakedefine", "#define")
            hout.write(line)
   return None

env["BUILDERS"]["GenerateConfig"] = Builder(action=GenerateConfig, suffix=".h", src_suffix=".h.in")

configh = env.GenerateConfig("blosc/config.h.in")

defs = ["USING_CMAKE"] # Force usage of config.h
incdirs = []
srcs = []

if cfg["lz4"]:
   incdirs.append("internal-complibs/lz4-1.7.2")
   srcs.extend(glob.glob("internal-complibs/lz4-1.7.2/*.c"))

if cfg["snappy"]:
   incdirs.append("internal-complibs/snappy-1.1.1")
   srcs.extend(glob.glob("internal-complibs/snappy-1.1.1/*.cc"))

if cfg["zlib"]:
   incdirs.append("internal-complibs/zlib-1.2.8")
   srcs.extend(glob.glob("internal-complibs/zlib-1.2.8/*.c"))

if cfg["zstd"]:
   incdirs.extend(["internal-complibs/zstd-1.0.0",
                   "internal-complibs/zstd-1.0.0/common"])
   srcs.extend(glob.glob("internal-complibs/zstd-1.0.0/common/*.c"))
   srcs.extend(glob.glob("internal-complibs/zstd-1.0.0/compress/*.c"))
   srcs.extend(glob.glob("internal-complibs/zstd-1.0.0/decompress/*.c"))
   srcs.extend(glob.glob("internal-complibs/zstd-1.0.0/dictBuilder/*.c"))

if cfg["sse2"]:
   defs.append("SHUFFLE_SSE2_ENABLED")
   srcs.extend(filter(lambda x: "-sse2" in x, glob.glob("blosc/*.c")))

if cfg["avx2"]:
   defs.append("SHUFFLE_AVX2_ENABLED")
   srcs.extend(filter(lambda x: "-avx2" in x, glob.glob("blosc/*.c")))

if not cfg["static"]:
   defs.append("BLOSC_SHARED_LIBRARY")

if sys.platform == "win32":
   defs.extend(["_CRT_NONSTDC_NO_DEPRECATE", "_CRT_SECURE_NO_WARNINGS"])

def CustomConfig(env):
   if cfg["sse2"]:
      if sys.platform == "win32":
         # On 64 bit arch, SSE2 is one by default
         #env.Append(CCFLAGS=" /arch:SSE2")
         pass
      else:
         env.Append(CCFLAGS=" -msse2")
   if cfg["avx2"]:
      if sys.platform == "win32":
         env.Append(CCFLAGS=" /arch:AVX2")
      else:
         env.Append(CCFLAGS=" -mavx2")
   if sys.platform not in ("win32", "darwin"):
      threads.Require(env)

def RequireBlosc(env):
   env.Append(LIBS=["blosc"])
   if not cfg["static"]:
      env.Append(CPPDFINES=["BLOSC_SHARED_LIBRARY"])
   else:
      threads.Require(env)

Export("RequireBlosc")

projs = [
   {
      "name": "blosc",
      "type": ("staticlib" if cfg["static"] else "sharedlib"),
      "bldprefix": ("static" if cfg["static"] else "shared"),
      "version": "1.11.1",
      "soname": "libblosc.so.1",
      "install_name": "libblosc.1.dylib",
      "defs": defs,
      "incdirs": incdirs,
      "srcs": filter(lambda x: "-sse2" not in x and "-avx2" not in x, glob.glob("blosc/*.c")) + srcs,
      "deps": configh,
      "custom": [CustomConfig],
      "install": {"include": ["blosc/blosc.h", "blosc/blosc-export.h"]}
   }
]

targets = excons.DeclareTargets(env, projs)
