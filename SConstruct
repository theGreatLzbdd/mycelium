#!/usr/bin/python /usr/bin/scons
# -*- coding: utf-8 -*-
import SCons.Errors
import sys
import os
from multiprocessing import cpu_count

import SCutils
import utils


############################################################################
# Commandline user options:
#
# Disable colorized output and produce full compiler messages
#
SCutils.add_option('noauto_j', 'disable automatic num_jobs detection', 0)
SCutils.add_option('verbose', 'verbose build', 0)
SCutils.add_option('colorblind', 'no colors', 0)
SCutils.add_option('build', 'specify build=(debug|release)', 1)
SCutils.add_option('ccache', 'ccache=(yes|no)', 1)
SCutils.add_option('installdir', 'installdir=path\tWhere to install the binaries to', 1)
SCutils.add_option('define', "Additional define", 1)
SCutils.add_option('assert', "enable assertions", 0)
SCutils.add_option('glibcxx_debug', "enable glibcxx debug", 0)
SCutils.add_option('gprof_profile', "compile with profiling support", 0)
SCutils.add_option('system_curl', "use system's curl", 0)

# Auto -j
if not SCutils.has_option('noauto_j'):
    PARALLELISM = int(cpu_count() * 1.15)
    print 'Autoselecting number of jobs: {0} (use --noauto_j to disable)'.format(PARALLELISM)
    SetOption('num_jobs', PARALLELISM)

else:
    PARALLELISM = 1

############################################################################



def bld_lex(target, source, env):
    print '%s -> %s' % (source[0],target[0])
    os.system('perl -0777 -pe \'s{/\*.*?\*/}{}gs\' < %s > %s' % (source[0],target[0]))
    return None


############################################################################


# Beyond this point we set up the compile environment
cppdefines = []


if SCutils.has_option('define'):
    cppdefines.append(SCutils.get_option('define'))

if SCutils.has_option('glibcxx_debug'):
    cppdefines.append('_GLIBCXX_DEBUG')
    cppdefines.append('_GLIBCXX_DEBUG_PEDANTIC')


ccflags = [
    '-Wall',
    '-march=native',
#    '-Wno-deprecated',
    '-Winvalid-pch',
#    '-I{0}'.format(Dir('3rd-party/boost/').get_abspath()),
    '-I{0}'.format(Dir('3rd_party/mongodb/src/mongo/').get_abspath())
]

linkflags = [
    '-rdynamic'
]

if SCutils.has_option('gprof_profile'):
    ccflags.append('-pg')
    linkflags.append('-pg')

for k, v in ARGLIST:
    if k == 'ccflag':
        ccflags.append(v)
        print '*****************\nAdding custom ccflag: {0}\nlist of current ccflags: '.format(v), ccflags,\
            '\n****************\n'

    elif k == 'define':
        cppdefines.append(v)
        print '****************\nAdding custom define: {0}\nlist of current defines: '.format(v), cppdefines,\
            '\n****************\n'

############################################################################
# Configure debug or release build environment
#
build = GetOption('build')
if not build:
    build = 'release'

#
# DEBUG
#
if build == 'debug':
    cppdefines.append('DEBUG')

    ccflags.extend([
        '-O0',
        '-ggdb3'
    ])

#
# RELEASE
#
elif build == 'release':
    if not SCutils.has_option('assert'):
        cppdefines.append('NDEBUG')


    ccflags.extend([
        '-O3'
    ])

else:
    raise SCons.Errors.StopError('build {0} not supported'.format(build))

env = Environment(
    platform = 'posix',
    CCFLAGS=ccflags,
    CPPDEFINES=cppdefines,
    LINKFLAGS=linkflags,
    tools=['default'],
    toolpath='.')

env.Decider('MD5-timestamp')


############################################################################
# Flex builder
flex_bld = env.Builder(action=bld_lex, suffix='.ll', src_suffix='.ll');
env.Append(BUILDERS={'Flex':flex_bld})
env.Append(LEXFLAGS=['-Cf'])

# The default scons LEXCOM doesn't correlate to line numbers in the C/C++ file
env['LEXCOM'] = '$LEX $LEXFLAGS --outfile=$TARGET $SOURCES'

############################################################################

############################################################################
# Build verbosity
if not SCutils.has_option('verbose'):
    SCutils.setup_quiet_build(env, True if SCutils.has_option('colorblind') else False)
############################################################################


############################################################################
# Look if ccache available or is explicitly disabled
if (GetOption('ccache')  and GetOption('ccache') == 'yes')\
    or (SCutils.which('ccache') and not GetOption('ccache') == 'no'):

    env['CC'] ='ccache gcc'
    env['CXX'] ='ccache g++'

############################################################################

includes = [
    '.'
]
libs = []
if not SCutils.has_option('system_curl'):
    print 'Linking with static curl'
    curl_static = File('3rd_party/curl_install/lib/libcurl.a')
    ares_static = File('3rd_party/c-ares_install/lib/libcares.a')
    mongoclient_static = File('3rd_party/mongodb/libmongoclient.a')
    libs.append(curl_static)
    libs.append(ares_static)
    libs.append(mongoclient_static)
    includes.append(Dir('3rd_party/curl_install/include/').get_abspath())
else:
    libs.append('curl')

libs.extend([
    'boost_filesystem',
    'z',
    'boost_system',
    'boost_regex',
    'log4cxx',
    'pthread',
    'event',
    'idn',
    'dl',
    'ssl',
    'crypto'
])




# Add the previous settings to the build environment

env.Append(CPPPATH=includes)
env.Append(LIBS=libs)
env.Append(LIBPATH=[Dir('lib')])
env.ParseConfig('pkg-config liblog4cxx --cflags --libs');

############################################################################
# Get the sources and store them in the environment

#VariantDir('build', '.')

env['sources'] = SCutils.get_sources('src/sources.txt')
env['unit_test_sources'] = utils.findall('src/unit_test', '*.cc', 1)
#print env['unit_test_sources']

SConscript('src/SConscript', exports=['env'],
    variant_dir='build/{0}'.format(build), duplicate=0)

############################################################################
# install target
#
INSTALL_PATH = None
if GetOption('installdir'):
    INSTALL_PATH = GetOption('installdir')
else:
    INSTALL_PATH = os.path.join(os.environ.get('HOME'), 'bin')

Alias('install', INSTALL_PATH)
env.Install(INSTALL_PATH, env['crawler'])

############################################################################
# doc target
#
env.Command('doc','doxygen.conf','doxygen doxygen.conf')

