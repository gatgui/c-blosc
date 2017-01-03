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
   "sse2": (excons.GetArgument("with-sse2", 1, int) != 0),
   "avx2": (excons.GetArgument("with-avx2", 0, int) != 0)
}

out_headers_dir = "%s/include" % excons.OutputBaseDirectory()

defs = []
incdirs = [out_headers_dir, "blosc"]
ccflags = []
srcs = filter(lambda x: "-sse2" not in x and "-avx2" not in x, glob.glob("blosc/*.c"))
customs = []

if cfg["lz4"]:
   defs.append("HAVE_LZ4")
   incdirs.append("internal-complibs/lz4-1.7.2")
   srcs.extend(glob.glob("internal-complibs/lz4-1.7.2/*.c"))

if cfg["snappy"]:
   defs.append("HAVE_SNAPPY")
   incdirs.append("internal-complibs/snappy-1.1.1")
   srcs.extend(glob.glob("internal-complibs/snappy-1.1.1/*.cc"))

if cfg["zlib"]:
   defs.append("HAVE_ZLIB")
   incdirs.append("internal-complibs/zlib-1.2.8")
   srcs.extend(glob.glob("internal-complibs/zlib-1.2.8/*.c"))

if cfg["zstd"]:
   defs.append("HAVE_ZSTD")
   incdirs.extend(["internal-complibs/zstd-1.0.0",
                   "internal-complibs/zstd-1.0.0/common"])
   srcs.extend(glob.glob("internal-complibs/zstd-1.0.0/common/*.c"))
   srcs.extend(glob.glob("internal-complibs/zstd-1.0.0/compress/*.c"))
   srcs.extend(glob.glob("internal-complibs/zstd-1.0.0/decompress/*.c"))
   srcs.extend(glob.glob("internal-complibs/zstd-1.0.0/dictBuilder/*.c"))

if cfg["sse2"]:
   defs.append("SHUFFLE_SSE2_ENABLED")
   if sys.platform == "win32":
      # On 64 bit arch, SSE2 is one by default
      #ccflags.append("/arch:SSE2")
      pass
   else:
      ccflags.append("-msse2")
   srcs.extend(filter(lambda x: "-sse2" in x, glob.glob("blosc/*.c")))

if cfg["avx2"]:
   defs.append("SHUFFLE_AVX2_ENABLED")
   if sys.platform == "win32":
      ccflags.append("/arch:AVX2")
   else:
      ccflags.append("-mavx2")
   srcs.extend(filter(lambda x: "-avx2" in x, glob.glob("blosc/*.c")))

if sys.platform == "win32":
   defs.extend(["_CRT_NONSTDC_NO_DEPRECATE", "_CRT_SECURE_NO_WARNINGS"])

def RequireBlosc(static=False):
   def _RealRequire(env):
      env.Append(LIBS=["blosc%s" % ("_s" if static else "")])
      if not static:
         env.Append(CPPDFINES=["BLOSC_SHARED_LIBRARY"])
      else:
         threads.Require(env)

   return _RealRequire

Export("RequireBlosc")

blosc_headers = env.Install(out_headers_dir, ["blosc/blosc.h", "blosc/blosc-export.h"])

projs = [
   {
      "name": "blosc_s",
      "type": "staticlib",
      "bldprefix": "static",
      "defs": defs,
      "ccflags": ccflags,
      "incdirs": incdirs,
      "srcs": srcs
   },
   {
      "name": "blosc",
      "type": "sharedlib",
      "bldprefix": "shared",
      "version": "1.11.1",
      "soname": "libblosc.so.1",
      "install_name": "libblosc.1.dylib",
      "defs": defs + ["BLOSC_SHARED_LIBRARY", "BLOSC_DLL_EXPORT"],
      "ccflags": ccflags,
      "incdirs": incdirs,
      "srcs": srcs,
      "custom": [threads.Require]
   }
]

excons.DeclareTargets(env, projs)
