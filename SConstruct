import re
import sys
import excons
import excons.tools.threads as threads


env = excons.MakeBaseEnv()

cfg = {
   "lz4": (excons.GetArgument("blosc-no-lz4", 0, int) == 0),
   "snappy": (excons.GetArgument("blosc-no-snappy", 0, int) == 0),
   "zlib": (excons.GetArgument("blosc-no-zlib", 0, int) == 0),
   "zstd": (excons.GetArgument("blosc-no-zstd", 0, int) == 0),
   "sse2": (excons.GetArgument("blosc-with-sse2", 1, int) != 0),
   "avx2": (excons.GetArgument("blosc-with-avx2", 0, int) != 0)
}

out_headers_dir = "%s/include" % excons.OutputBaseDirectory()

defs = []
incdirs = [out_headers_dir, "blosc"]
ccflags = []
srcs = filter(lambda x: "-sse2" not in x and "-avx2" not in x, excons.glob("blosc/*.c"))
customs = []

if cfg["lz4"]:
   defs.append("HAVE_LZ4")
   incdirs.append("internal-complibs/lz4-1.7.5")
   srcs.extend(excons.glob("internal-complibs/lz4-1.7.5/*.c"))

if cfg["snappy"]:
   defs.append("HAVE_SNAPPY")
   incdirs.append("internal-complibs/snappy-1.1.1")
   srcs.extend(excons.glob("internal-complibs/snappy-1.1.1/*.cc"))

if cfg["zlib"]:
   defs.append("HAVE_ZLIB")
   incdirs.append("internal-complibs/zlib-1.2.8")
   srcs.extend(excons.glob("internal-complibs/zlib-1.2.8/*.c"))

if cfg["zstd"]:
   defs.append("HAVE_ZSTD")
   incdirs.extend(["internal-complibs/zstd-1.1.4",
                   "internal-complibs/zstd-1.1.4/common"])
   srcs.extend(excons.glob("internal-complibs/zstd-1.1.4/common/*.c"))
   srcs.extend(excons.glob("internal-complibs/zstd-1.1.4/compress/*.c"))
   srcs.extend(excons.glob("internal-complibs/zstd-1.1.4/decompress/*.c"))
   srcs.extend(excons.glob("internal-complibs/zstd-1.1.4/dictBuilder/*.c"))

if cfg["sse2"]:
   defs.append("SHUFFLE_SSE2_ENABLED")
   if sys.platform == "win32":
      # On 64 bit arch, SSE2 is one by default
      #ccflags.append("/arch:SSE2")
      pass
   else:
      ccflags.append("-msse2")
   srcs.extend(filter(lambda x: "-sse2" in x, excons.glob("blosc/*.c")))

if cfg["avx2"]:
   defs.append("SHUFFLE_AVX2_ENABLED")
   if sys.platform == "win32":
      ccflags.append("/arch:AVX2")
   else:
      ccflags.append("-mavx2")
   srcs.extend(filter(lambda x: "-avx2" in x, excons.glob("blosc/*.c")))

if sys.platform == "win32":
   defs.extend(["_CRT_NONSTDC_NO_DEPRECATE", "_CRT_SECURE_NO_WARNINGS"])


build_opts = """BLOSC OPTIONS
   no-lz4=0|1    : Disable lz4 support    [0]
   no-snappy=0|1 : Disable snappy support [0]
   no-zlib=0|1   : Disable zlib support   [0]
   no-zstd=0|1   : Disable zstd support   [0]
   with-sse2=0|1 : Enable SSE2 support    [1]
   with-avx2=0|1 : Enable AVX2 support    [0]"""

def BloscName(static=False):
   return "blosc%s" % ("_s" if static else "")

def BloscPath(static=False):
   name = BloscName(static)
   if sys.platform == "win32":
      libname = name + ".lib"
   else:
      libname = "lib" + name + (".a" if static else excons.SharedLibraryLinkExt())
   return excons.OutputBaseDirectory() + "/lib/" + libname

def RequireBlosc(env, static=False):
   if not static:
      env.Append(CPPDFINES=["BLOSC_SHARED_LIBRARY"])
   env.Append(CPPPATH=[excons.OutputBaseDirectory() + "/include"])
   env.Append(CPPPATH=[excons.OutputBaseDirectory() + "/lib"])
   excons.Link(env, BloscName(static), static=static, force=True, silent=True)
   if static:
      threads.Require(env)

Export("BloscName BloscPath RequireBlosc")

blosc_headers = env.Install(out_headers_dir, ["blosc/blosc.h", "blosc/blosc-export.h"])

projs = [
   {
      "name": "blosc_s",
      "type": "staticlib",
      "desc": "Blosc static library",
      "bldprefix": "static",
      "defs": defs,
      "ccflags": ccflags,
      "incdirs": incdirs,
      "srcs": srcs
   },
   {
      "name": "blosc",
      "type": "sharedlib",
      "desc": "Blosc shared library",
      "bldprefix": "shared",
      "version": "1.11.3",
      "soname": "libblosc.so.1",
      "install_name": "libblosc.1.dylib",
      "defs": defs + ["BLOSC_SHARED_LIBRARY", "BLOSC_DLL_EXPORT"],
      "ccflags": ccflags,
      "incdirs": incdirs,
      "srcs": srcs,
      "custom": [threads.Require]
   }
]

excons.AddHelpOptions(blosc=build_opts)

targets = excons.DeclareTargets(env, projs)

env.Depends(targets["blosc"], blosc_headers)
env.Depends(targets["blosc_s"], blosc_headers)
