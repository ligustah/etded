# -*- mode: python -*-
# ET build script
# TTimo <ttimo@idsoftware.com>
# http://scons.sourceforge.net

import sys, os, time, commands, re, pickle, StringIO, popen2, commands, pdb, zipfile, string, tempfile
import SCons

import scons_utils

conf_filename='site.conf'
# choose configuration variables which should be saved between runs
# ( we handle all those as strings )
serialized=[ 'CC', 'CXX', 'JOBS', 'BUILD', 'BUILD_ROOT', 'MASTER', 'NOCONF' ]

# help -------------------------------------------

Help("""
Usage: scons [OPTIONS] [TARGET] [CONFIG]

[OPTIONS] and [TARGET] are covered in command line options, use scons -H

[CONFIG]: KEY="VALUE" [...]
a number of configuration options saved between runs in the """ + conf_filename + """ file
erase """ + conf_filename + """ to start with default settings again

CC (default gcc)
CXX (default g++)
	Specify C and C++ compilers (defaults gcc and g++)
	ex: CC="gcc-3.3"
	You can use ccache and distcc, for instance:
	CC="ccache distcc gcc" CXX="ccache distcc g++"

JOBS (default 1)
	Parallel build

BUILD (default debug)
	Use debug-all/debug/release to select build settings
	ex: BUILD="release"
	debug-all: no optimisations, debugging symbols
	debug: -O -g
	release: all optimisations, including CPU target etc.

BUILD_ROOT (default 'build')
	change the build root directory
	
NOCONF (default 0, not saved)
	ignore site configuration and use defaults + command line only

MASTER (default '')
	set to override the default master server address
"""
)

# end help ---------------------------------------

# sanity -----------------------------------------

EnsureSConsVersion( 0, 96 )

# end sanity -------------------------------------

# system detection -------------------------------

# CPU type
cpu = commands.getoutput('uname -m')
dll_cpu = '???' # grmbl, alternative naming for .so
exp = re.compile('.*i?86.*')
if exp.match(cpu):
	cpu = 'x86'
	dll_cpu = 'i386'
else:
	cpu = commands.getoutput('uname -p')
	if ( cpu == 'powerpc' ):
		cpu = 'ppc'
		dll_cpu = cpu
	else:
		cpu = 'cpu'
		dll_cpu = cpu
OS = commands.getoutput( 'uname -s' )
if ( OS == 'Darwin' ):
	print 'EXPERIMENTAL - INCOMPLETE'
	print 'Use the XCode projects to compile ET universal binaries for OSX'
	cpu = 'osx'
	dll_cpu = 'osx'

print 'cpu: ' + cpu

# end system detection ---------------------------

# default settings -------------------------------

CC = 'gcc -m32'
CXX = 'g++ -m32'
JOBS = '1'
BUILD = 'debug'
BUILD_ROOT = 'build'
NOCONF = '0'
MASTER = ''


# end default settings ---------------------------

# arch detection ---------------------------------

def gcc_major():
	major = os.popen( CC + ' -dumpversion' ).read().strip()
	major = re.sub('^([^.]+)\\..*$', '\\1', major)
	print 'gcc major: %s' % major
	return major

gcc3 = (gcc_major() != '2')

def gcc_is_mingw():
	mingw = os.popen( CC + ' -dumpmachine' ).read()
	return re.search('mingw', mingw) != None

if gcc_is_mingw():
	g_os = 'win32'
elif OS == 'Darwin':
	g_os = 'Darwin'
else:
	g_os = 'Linux'
print 'OS: %s' % g_os

# end arch detection -----------------------------

# site settings ----------------------------------

if ( not ARGUMENTS.has_key( 'NOCONF' ) or ARGUMENTS['NOCONF'] != '1' ):
	site_dict = {}
	if (os.path.exists(conf_filename)):
		site_file = open(conf_filename, 'r')
		p = pickle.Unpickler(site_file)
		site_dict = p.load()
		print 'Loading build configuration from ' + conf_filename + ':'
		for k, v in site_dict.items():
			exec_cmd = k + '=\'' + v + '\''
			print '  ' + exec_cmd
			exec(exec_cmd)
else:
	print 'Site settings ignored'

# end site settings ------------------------------

# command line settings --------------------------

for k in ARGUMENTS.keys():
	exec_cmd = k + '=\'' + ARGUMENTS[k] + '\''
	print 'Command line: ' + exec_cmd
	exec( exec_cmd )

# end command line settings ----------------------

# save site configuration ----------------------

if ( not ARGUMENTS.has_key( 'NOCONF' ) or ARGUMENTS['NOCONF'] != '1' ):
	for k in serialized:
		exec_cmd = 'site_dict[\'' + k + '\'] = ' + k
		exec(exec_cmd)

	site_file = open(conf_filename, 'w')
	p = pickle.Pickler(site_file)
	p.dump(site_dict)
	site_file.close()

# end save site configuration ------------------

# general configuration, target selection --------

g_build = BUILD_ROOT + '/' + BUILD

SConsignFile( 'scons.signatures' )

SetOption('num_jobs', JOBS)

if ( OS == 'Linux' ):
	LINK = CC
else:
	LINK = CXX

# common flags
# BASE + CORE + OPT for engine
# BASE + GAME + OPT for game

BASECPPFLAGS = [ ]
CORECPPPATH = [ ]
CORELIBPATH = [ ]
CORECPPFLAGS = [ ]
GAMECPPFLAGS = [ ]
BASELINKFLAGS = [ ]
CORELINKFLAGS = [ ]

# for release build, further optimisations that may not work on all files
OPTCPPFLAGS = [ ]

if ( OS == 'Darwin' ):
	BASECPPFLAGS += [ '-D__MACOS__', '-Wno-long-double', '-Wno-unknown-pragmas', '-Wno-trigraphs', '-fpascal-strings', '-fasm-blocks', '-Wreturn-type', '-Wunused-variable', '-ffast-math', '-fno-unsafe-math-optimizations', '-fvisibility=hidden', '-mmacosx-version-min=10.4', '-isysroot', '/Developer/SDKs/MacOSX10.4u.sdk' ]
	BASECPPFLAGS += [ '-arch', 'i386' ]
	BASELINKFLAGS += [ '-arch', 'i386' ]

BASECPPFLAGS.append( '-pipe' )
# warn all
BASECPPFLAGS.append( '-Wall' )
# don't wrap gcc messages
BASECPPFLAGS.append( '-fmessage-length=0' )

if ( BUILD == 'debug-all' ):
	BASECPPFLAGS.append( '-g' )
	BASECPPFLAGS.append( '-D_DEBUG' )
elif ( BUILD == 'debug' ):
	BASECPPFLAGS.append( '-g' )
	BASECPPFLAGS.append( '-O1' )
	BASECPPFLAGS.append( '-D_DEBUG' )
elif ( BUILD == 'release' ):
	BASECPPFLAGS.append( '-DNDEBUG' )
	if ( OS == 'Linux' ):
		# -fomit-frame-pointer: gcc manual indicates -O sets this implicitely,
		# only if that doesn't affect debugging
		# on Linux, this affects backtrace capability, so I'm assuming this is needed
		# -finline-functions: implicit at -O3
		# -fschedule-insns2: implicit at -O3
		# -funroll-loops ?
		# -mfpmath=sse -msse ?
		OPTCPPFLAGS = [ '-O3', '-march=i686', '-Winline', '-ffast-math', '-fomit-frame-pointer', '-finline-functions', '-fschedule-insns2' ]
	elif ( OS == 'Darwin' ):
		OPTCPPFLAGS = []
else:
	print 'Unknown build configuration ' + BUILD
	sys.exit(0)

# create the build environements
if g_os == 'win32':
	g_base_env = Environment( tools=['mingw'], ENV = os.environ, CC = CC, CXX = CXX, LINK = LINK, CPPFLAGS = BASECPPFLAGS, LINKFLAGS = BASELINKFLAGS, CPPPATH = CORECPPPATH, LIBPATH = CORELIBPATH )
else:
	g_base_env = Environment( ENV = os.environ, CC = CC, CXX = CXX, LINK = LINK, CPPFLAGS = BASECPPFLAGS, LINKFLAGS = BASELINKFLAGS, CPPPATH = CORECPPPATH, LIBPATH = CORELIBPATH )
scons_utils.SetupUtils( g_base_env )

g_env = g_base_env.Clone()

g_env['CPPFLAGS'] += OPTCPPFLAGS
g_env['CPPFLAGS'] += CORECPPFLAGS
g_env['LINKFLAGS'] += CORELINKFLAGS

if ( OS == 'Darwin' ):
	# configure for dynamic bundle
	g_env['SHLINKFLAGS'] = '$LINKFLAGS -bundle -flat_namespace -undefined suppress'
	g_env['SHLIBSUFFIX'] = '.so'

# maintain this dangerous optimization off at all times
g_env.Append( CPPFLAGS = '-fno-strict-aliasing' )

if ( int(JOBS) > 1 ):
	print 'Using buffered process output'
	scons_utils.SetupBufferedOutput( g_env )

# mark the globals

local_dedicated = 0
# curl
local_curl = 0	# config selection
curl_lib = []

GLOBALS = 'g_env OS g_os BUILD local_dedicated curl_lib local_curl MASTER gcc3 cpu'

# end general configuration ----------------------

# win32 cross compilation ----------------------

if g_os == 'win32' and os.name != 'nt':
	# mingw doesn't define the cpu type, but our only target is x86
	g_env.Append(CPPDEFINES = '_M_IX86=400')
	g_env.Append(LINKFLAGS = '-static-libgcc')
	# scons doesn't support cross-compiling, so set these up manually
	g_env['STATIC_AND_SHARED_OBJECTS_ARE_THE_SAME'] = 1
	g_env['WIN32DEFSUFFIX']	= '.def'
	g_env['PROGSUFFIX']	= '.exe'
	g_env['SHLIBSUFFIX']	= '.dll'
	g_env['SHCCFLAGS']	= '$CCFLAGS'
	cpu = 'x86'

# end win32 cross compilation ------------------

# targets ----------------------------------------

toplevel_targets = []

# build curl if needed
#if ( OS != 'Darwin' ):
#	# 1: debug, 2: release
#	if ( BUILD == 'release' ):
#		local_curl = 2
#	else:
#		local_curl = 1
#	Export( 'GLOBALS ' + GLOBALS )
#	curl_lib = SConscript( 'SConscript.curl' )

local_dedicated = 1
Export( 'GLOBALS ' + GLOBALS )
BuildDir( g_build + '/dedicated', '.', duplicate = 0 )
etded = SConscript( g_build + '/dedicated/SConscript.core' )
if ( g_os == 'win32' ):
	toplevel_targets.append( InstallAs( '#etded.exe', etded ) )
else:
	toplevel_targets.append( InstallAs( '#etded.' + cpu, etded ) )

# end targets ------------------------------------
