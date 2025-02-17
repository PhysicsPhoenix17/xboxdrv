# -*- python -*-

import os
import subprocess
import string
import re

def build_dbus_glue(target, source, env):
    """
    C++ doesn't allow casting from void* to a function pointer,
    thus we have to change the code to use a union to do the
    conversion.
    """
    xml = subprocess.Popen(["dbus-binding-tool",
                            "--mode=glib-server",
                            "--prefix=" + env['DBUS_PREFIX'], source[0].get_path()],
                           stdout=subprocess.PIPE).communicate()[0]

    xml = re.sub(r"callback = \(([A-Za-z_]+)\) \(marshal_data \? marshal_data : cc->callback\);",
                 r"union { \1 fn; void* obj; } conv;\n  "
                 "conv.obj = (marshal_data ? marshal_data : cc->callback);\n  "
                 "callback = conv.fn;", xml.decode('utf-8'))

    with open(target[0].get_path(), "w") as f:
        f.write(xml)

def build_bin2h(target, source, env):
    """
    Takes a list of files and converts them into a C source that can be included
    """
    def c_escape(str): 
        return str.translate(str.maketrans("/.-", "___"))
    
    print(target)
    print(source)
    with open(target[0].get_path(), "w") as fout:
        fout.write("// autogenerated by scons Bin2H builder, do not edit by hand!\n\n")

        if env.has_key("BIN2H_NAMESPACE"):
            fout.write("namespace %s {\n\n" % env["BIN2H_NAMESPACE"])
            
        # write down data
        for src in source:
            with open(src.get_path(), "rb") as fin:
                data = fin.read()
                fout.write("// \"%s\"\n" % src.get_path())
                fout.write("const char %s[] = {" % c_escape(src.get_path()))
                bytes_arr = ["0x%02x" % c for c in data]
                for i in range(len(bytes_arr)):
                    if i % 13 == 0:
                        fout.write("\n  ")
                    fout.write(bytes_arr[i])
                    if i != len(bytes_arr)-1:
                        fout.write(", ")
                fout.write("\n};\n\n")

        # write down file table
        if False:
            fout.write("const char** file_table = {\n")
            fout.write(string.join(["  %-35s %-s" % ("\"%s\"," % src.get_path(),
                                                     c_escape(src.get_path()))
                                    for src in source], ",\n"))
            fout.write("\n}\n\n")

        if env.has_key("BIN2H_NAMESPACE"):
            fout.write("} // namespace %s\n\n" % env["BIN2H_NAMESPACE"])
                
        fout.write("/* EOF */\n")
                

env = Environment(ENV=os.environ, BUILDERS = {
    'DBusGlue' : Builder(action = build_dbus_glue),
    'Bin2H'    : Builder(action = build_bin2h)
    })

opts = Variables(['custom.py'], ARGUMENTS)

opts.Add('CPPPATH', 'Additional preprocessor paths')
opts.Add('CPPFLAGS', 'Additional preprocessor flags')
opts.Add('CPPDEFINES', 'defined constants')
opts.Add('LIBPATH', 'Additional library paths')
opts.Add('LIBS', 'Additional libraries')
opts.Add('CCFLAGS', 'C Compiler flags')
opts.Add('CXXFLAGS', 'C++ Compiler flags')
opts.Add('LINKFLAGS', 'Linker Compiler flags')
opts.Add('AR', 'Library archiver')
opts.Add('CC', 'C Compiler')
opts.Add('CXX', 'C++ Compiler')
opts.Add('BUILD', 'Build type: release, custom, development')
opts.Add('PKG_CONFIG', 'pkg-config helper tool', 'pkg-config')

opts.Update(env)
Help(opts.GenerateHelpText(env))

env.Append(CPPPATH=["src/"])

if 'BUILD' in env and env['BUILD'] == 'development':
    env.Append(CXXFLAGS = [ "-O3",
                            "-g3",
                            "-ansi",
                            "-pedantic",
                            "-Wall",
                            "-Wextra",
                            "-Werror",
                            "-Wnon-virtual-dtor",
                            "-Weffc++",
                            # "-Wunreachable-code",
                            # "-Wconversion",
                            "-Wold-style-cast",
                            "-Wshadow",
                            "-Wcast-qual",
                            "-Winit-self", # only works with >= -O1
                            "-Wno-unused-parameter"])
elif 'BUILD' in env and env['BUILD'] == 'custom':
    pass
else:
    env.Append(CPPFLAGS = ['-g', '-O3', '-Wall', '-ansi', '-pedantic'])

env.ParseConfig(env['PKG_CONFIG'] + " --cflags --libs dbus-glib-1 | sed 's/-I/-isystem/g'")
env.ParseConfig(env['PKG_CONFIG'] + " --cflags --libs glib-2.0 | sed 's/-I/-isystem/g'")
env.ParseConfig(env['PKG_CONFIG'] + " --cflags --libs gthread-2.0 | sed 's/-I/-isystem/g'")
env.ParseConfig(env['PKG_CONFIG'] + " --cflags --libs libusb-1.0 | sed 's/-I/-isystem/g'")
env.ParseConfig(env['PKG_CONFIG'] + " --cflags --libs libudev | sed 's/-I/-isystem/g'")

f = open("VERSION")
package_version = f.read()
f.close()
    
env.Append(CPPDEFINES = { 'PACKAGE_VERSION': "'\"%s\"'" % package_version })

conf = Configure(env)

if not conf.env['CXX']:
    print("g++ must be installed!")
    Exit(1)

# X11 checks
if not conf.CheckLibWithHeader('X11', 'X11/Xlib.h', 'C++'):
    print('libx11-dev must be installed!')
    Exit(1)

env = conf.Finish()

env.Bin2H("src/xboxdrv_vfs.hpp", [
    "examples/mouse.xboxdrv",
    "examples/xpad-wireless.xboxdrv"
    ],
          BIN2H_NAMESPACE="xboxdrv_vfs")
env.DBusGlue("src/xboxdrv_daemon_glue.hpp",     "src/xboxdrv_daemon.xml", DBUS_PREFIX="xboxdrv_daemon")
env.DBusGlue("src/xboxdrv_controller_glue.hpp", "src/xboxdrv_controller.xml", DBUS_PREFIX="xboxdrv_controller")

libxboxdrv = env.StaticLibrary('xboxdrv',
                               Glob('src/*.cpp') +
                               Glob('src/axisfilter/*.cpp') +
                               Glob('src/buttonfilter/*.cpp') +
                               Glob('src/axisevent/*.cpp') +
                               Glob('src/buttonevent/*.cpp') +
                               Glob('src/modifier/*.cpp'))
env.Prepend(LIBS = libxboxdrv)

for file in Glob('test/*_test.cpp', strings=True):
    Alias('tests', env.Program(file[:-4], file))

Default(env.Program('xboxdrv', Glob('src/main/main.cpp')))

# EOF #
