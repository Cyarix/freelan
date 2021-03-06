"""
The SConstruct file.

Works on all UNIX-like operating systems.
"""

import os
import sys

from fnmatch import fnmatch

# This file is local.
from defines import Defines

AddOption(
    '--mode',
    dest='mode',
    nargs=1,
    action='store',
    choices=('all', 'debug', 'release'),
    default='all',
    help='The compilation mode.',
)


class FreelanEnvironment(Environment):
    """
    A freelan specific environment class.
    """

    def __init__(self, mode, prefix, bin_prefix=None, **kwargs):
        """
        Initialize the environment.

        :param mode: The compilation mode.
        :param prefix: The installation prefix.
        """
        super(FreelanEnvironment, self).__init__(**kwargs)

        # Inherit the environment from the context.
        self['ENV'] = os.environ

        self.defines = Defines()
        self.defines.register_into(self)

        for flag in [
            'CC',
            'CXX',
            'AR',
            'LINK',
        ]:
            if flag in os.environ:
                self[flag] = os.environ[flag]

        for flag in [
            'CFLAGS',
            'CXXFLAGS',
            'CPPFLAGS',
            'ARFLAGS',
            'LDFLAGS',
            'LINKFLAGS',
            'LIBS',
        ]:
            if flag in os.environ:
                self[flag] = Split(os.environ[flag])

        self.mode = mode
        self.prefix = prefix
        self.bin_prefix = bin_prefix if bin_prefix else prefix
        self.destdir = self['ENV'].get('DESTDIR', '')

        if self.destdir:
            self.install_prefix = os.path.normpath(
                os.path.abspath(self.destdir),
            ) + self.prefix
            self.bin_install_prefix = os.path.normpath(
                os.path.abspath(self.destdir),
            ) + self.bin_prefix
        else:
            self.install_prefix = self.prefix
            self.bin_install_prefix = self.bin_prefix

        if os.path.basename(self['CXX']) == 'clang++':
            self.Append(CXXFLAGS=['-Qunused-arguments'])
            self.Append(CXXFLAGS=['-fcolor-diagnostics'])
        elif os.path.basename(self['CXX']).startswith('g++'):
            self.Append(CXXFLAGS=['-Wno-missing-field-initializers'])

        self.Append(CXXFLAGS=['--std=c++11'])
        self.Append(CXXFLAGS=['-Wall'])
        self.Append(CXXFLAGS=['-Wextra'])
        self.Append(CXXFLAGS=['-Werror'])
        self.Append(CXXFLAGS=['-pedantic'])
        self.Append(CXXFLAGS=['-Wshadow'])
        self.Append(LDFLAGS=['--std=c++11'])

        if sys.platform.startswith('darwin'):
            self.Append(CXXFLAGS=['-arch', 'x86_64'])
            self.Append(CXXFLAGS=['-DBOOST_ASIO_DISABLE_KQUEUE'])
            self.Append(CXXFLAGS=['--stdlib=libc++'])
            self.Append(LDFLAGS=['--stdlib=libc++'])

        if self.mode == 'debug':
            self.Append(CXXFLAGS=['-g'])
            self.Append(CXXFLAGS='-DFREELAN_DEBUG=1')
        else:
            self.Append(CXXFLAGS='-O3')

        self.Append(CPPDEFINES=r'FREELAN_INSTALL_PREFIX="\"%s\""' % self.prefix)

    def RGlob(self, path, patterns=None):
        """
        Returns a list of file objects that match the specified patterns.
        """

        path = unicode(path)

        if isinstance(patterns, basestring):
            patterns = [patterns]

        result = []
        abspath = Dir(path).srcnode().abspath

        for root, ds, fs in os.walk(abspath):
            prefix = os.path.relpath(root, abspath)
            for f in fs:
                if not patterns or any(fnmatch(f, pattern) for pattern in patterns):
                    result.append(File(os.path.join(path, prefix, f)))

        return result

    def RInstall(self, target, source, patterns=None):
        """
        Install a directory, keeping its structure.
        """

        files = self.RGlob(source, patterns)
        result = []

        for f in files:
            result.append(self.Install(os.path.join(str(target), os.path.dirname(str(f))), f))

        return result

    def SymLink(self, target, source):
        def create_symlink(target, source, env):
            os.symlink(os.path.abspath(str(source[0])), os.path.abspath(str(target[0])))

        return self.Command(target, source, create_symlink)


mode = GetOption('mode')
prefix = os.path.normpath(os.path.abspath(ARGUMENTS.get('prefix', './install')))

if 'bin_prefix' in ARGUMENTS:
    bin_prefix = os.path.normpath(os.path.abspath(ARGUMENTS['bin_prefix']))
else:
    bin_prefix = None

if mode in ('all', 'release'):
    env = FreelanEnvironment(mode='release', prefix=prefix, bin_prefix=bin_prefix)
    libraries, includes, apps, samples, configurations = SConscript('SConscript', exports='env', variant_dir=os.path.join('build', env.mode))
    install = env.Install(os.path.join(env.bin_install_prefix, 'bin'), apps)
    install.extend(env.Install(os.path.join(env.install_prefix, 'etc', 'freelan'), configurations))

    Alias('install', install)
    Alias('apps', apps)
    Alias('samples', samples)
    Alias('all', install + apps + samples)

if mode in ('all', 'debug'):
    env = FreelanEnvironment(mode='debug', prefix=prefix)
    libraries, includes, apps, samples, configurations = SConscript('SConscript', exports='env', variant_dir=os.path.join('build', env.mode))
    Alias('apps', apps)
    Alias('samples', samples)
    Alias('all', apps + samples)

if sys.platform.startswith('darwin'):
    retail_prefix = '/usr/local'
    env = FreelanEnvironment(mode='retail', prefix=retail_prefix)
    libraries, includes, apps, samples, configurations = SConscript('SConscript', exports='env', variant_dir=os.path.join('build', env.mode))
    package = SConscript('packaging/osx/SConscript', exports='env apps configurations retail_prefix')
    install_package = env.Install('.', package)
    Alias('package', install_package)

Default('install')
