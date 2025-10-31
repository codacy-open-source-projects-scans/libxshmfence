# SPDX-License-Identifier: MIT
#
# Copyright (c) 2025, Oracle and/or its affiliates.
#
# Permission is hereby granted, free of charge, to any person obtaining a
# copy of this software and associated documentation files (the "Software"),
# to deal in the Software without restriction, including without limitation
# the rights to use, copy, modify, merge, publish, distribute, sublicense,
# and/or sell copies of the Software, and to permit persons to whom the
# Software is furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice (including the next
# paragraph) shall be included in all copies or substantial portions of the
# Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.  IN NO EVENT SHALL
# THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
# FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
# DEALINGS IN THE SOFTWARE.
#


project(
  'libxshmfence',
  'c',
  version: '1.3.3',
  license: 'HPND-sell-variant',
  license_files: 'COPYING',
  meson_version: '>= 1.3.0',
)

# Replacement for XORG_DEFAULT_OPTIONS
cc = meson.get_compiler('c')
if cc.has_argument('-fno-strict-aliasing')
  add_project_arguments('-fno-strict-aliasing', language: 'c')
endif

conf = configuration_data()

# Replaces AC_USE_SYSTEM_EXTENSIONS
if host_machine.system() == 'sunos'
    system_extensions = '__EXTENSIONS__'
else
    system_extensions = '_GNU_SOURCE'
endif
conf.set(system_extensions, 1,
         description: 'Enable non-standardized system API extensions')

futex = false
if cc.has_header('linux/futex.h', required: get_option('futex'))
  futex = true
elif cc.has_header('sys/umtx.h', required: get_option('futex'),
                   prefix: ['#include <errno.h>', '#include <sys/types.h>'])
  futex = true
  conf.set('HAVE_UMTX', 1, description: 'Use umtx')
endif

if futex
  pthread = false
  conf.set('HAVE_FUTEX', 1, description: 'Use futexes')
else
  pthread = true
  conf.set('HAVE_PTHREAD', 1, description: 'Use pthread primitives')
endif
summary('Synchronization method', futex ? 'futexes' : 'pthread primitives')

if pthread
  thread_dep = cc.find_library('pthread')
  cc.has_function('pthread_create', dependencies: thread_dep,
                  prefix: '#include <pthread.h>', required: true)
else
  thread_dep = dependency('', required: false)
endif

xproto_dep = dependency('xproto', required: true)

if cc.has_function('memfd_create', required: false,
                   prefix: ['#define ' + system_extensions,
                            '#include <sys/mman.h>'])
  conf.set('HAVE_MEMFD_CREATE', 1)
endif

if cc.has_function('mkostemp', required: false,
                   prefix: ['#define ' + system_extensions,
                            '#include <stdlib.h>', '#include <fcntl.h>'])
  conf.set('HAVE_MKOSTEMP', 1)
endif

if cc.has_header_symbol('<asm/unistd.h>', '__NR_memfd_create', required: false,
                        prefix: '#define ' + system_extensions)
  conf.set('HAVE_DECL___NR_MEMFD_CREATE', 1)
endif

if cc.has_header('sys/memfd.h', required: false)
  conf.set('HAVE_MEMFD_H', 1, description: 'Has sys/memfd.h header')
endif

if get_option('visibility').allowed()
  visibility = 'hidden'
else
  visibility = 'default'
endif
summary('Symbol visibility', visibility)

#
# Locate a suitable tmp file system for creating shared memory files
#

shmdir = get_option('shared-memory-dir')
if shmdir == 'auto'
  if meson.is_cross_build()
    error('Must specify -Dshared-memory-dir=<path> when cross-compiling')
  endif
  shmdirs=['/run/shm', '/dev/shm', '/var/tmp', '/tmp']
  fs = import('fs')
  foreach dir: shmdirs
    if fs.is_dir(dir)
      shmdir = dir
      break
    endif
  endforeach
  if shmdir == 'auto'
    error('No directory found for shared memory temp files.')
  endif
elif shmdir.startswith('/') == false
  error('Invalid directory specified for -Dshared-memory-dir:', shmdir)
endif

conf.set_quoted('SHMDIR', shmdir,
                description: 'Directory for shared memory temp files')
summary('Shared memory dir', shmdir)                

configure_file(output: 'config.h', configuration: conf)
add_project_arguments('-DHAVE_CONFIG_H', language : 'c')

libxshmfence_sources = ['src/xshmfence_alloc.c']
if pthread
  libxshmfence_sources += 'src/xshmfence_pthread.c'
endif
if futex
  libxshmfence_sources += 'src/xshmfence_futex.c'
endif

libxshmfence = library(
  'xshmfence',
  libxshmfence_sources,
  dependencies: [thread_dep, xproto_dep],
  gnu_symbol_visibility: visibility,
  include_directories: 'src',
  version: '1.0.0',
  install: true,
)

install_headers(
  'src/xshmfence.h',
  subdir: 'X11',
)

xshmfence_test = executable(
  'xshmfence_test',
  'test/xshmfence_test.c',
  include_directories: 'src',
  dependencies: thread_dep,
  link_with: libxshmfence,
)
test('xshmfence_test', xshmfence_test)

pkg = import('pkgconfig')
pkg.generate(
  libxshmfence,
  description: 'The X Shared Memory Fence Library',
  filebase: 'xshmfence',
  url: 'https://gitlab.freedesktop.org/xorg/lib/libxshmfence',
  requires: 'xproto'
)
