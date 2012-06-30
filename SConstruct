#!/usr/bin/python

# scons script for the Arduino sketch
# http://github.com/suapapa/arscons
#
# Copyright (C) 2010-2012 by Homin Lee <homin.lee@suapapa.net>
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; either version 3 of the License, or
# (at your option) any later version.

# You'll need the serial module: http://pypi.python.org/pypi/pyserial

# Basic Usage:
# 1. make a folder which have same name of the sketch (ex. Blink/ for Blink.pde)
# 2. put the sketch and SConstruct(this file) under the folder.
# 3. to make the HEX. do following in the folder.
#     $ scons
# 4. to upload the binary, do following in the folder.
#     $ scons upload

# Thanks to:
# * Ovidiu Predescu <ovidiu@gmail.com> and Lee Pike <leepike@gmail.com>
#     for Mac port and bugfix.
#
# This script tries to determine the port to which you have an Arduino
# attached. If multiple USB serial devices are attached to your
# computer, you'll need to explicitly specify the port to use, like
# this:
#
# $ scons ARDUINO_PORT=/dev/ttyUSB0
#
# To add your own directory containing user libraries, pass EXTRA_LIB
# to scons, like this:
#
# $ scons EXTRA_LIB=<my-extra-library-dir>
#

from glob import glob
from itertools import ifilter, imap
from subprocess import check_call, CalledProcessError
import sys
import re
import os
pathJoin = os.path.join

env = Environment()
platform = env['PLATFORM']

def getUsbTty(rx):
    usb_ttys = glob(rx)
    return usb_ttys[0] if len(usb_ttys) == 1 else None

AVR_BIN_PREFIX = None
AVRDUDE_CONF = None

if platform == 'darwin':
    # For MacOS X, pick up the AVR tools from within Arduino.app
    ARDUINO_HOME_DEFAULT = '/Applications/Arduino.app/Contents/Resources/Java'
    ARDUINO_PORT_DEFAULT = getUsbTty('/dev/tty.usbserial*')
    SKETCHBOOK_HOME_DEFAULT = ''
elif platform == 'win32':
    # For Windows, use environment variables.
    ARDUINO_HOME_DEFAULT = os.environ.get('ARDUINO_HOME')
    ARDUINO_PORT_DEFAULT = os.environ.get('ARDUINO_PORT')
    SKETCHBOOK_HOME_DEFAULT = ''
else:
    # For Ubuntu Linux (9.10 or higher)
    ARDUINO_HOME_DEFAULT = '/usr/share/arduino/' #'/home/YOU/apps/arduino-00XX/'
    ARDUINO_PORT_DEFAULT = getUsbTty('/dev/ttyUSB*')
    AVR_BIN_PREFIX = 'avr-'
    SKETCHBOOK_HOME_DEFAULT = os.path.realpath('~/share/arduino/sketchbook/')

ARDUINO_BOARD_DEFAULT = os.environ.get('ARDUINO_BOARD', 'atmega328')

ARDUINO_HOME    = ARGUMENTS.get('ARDUINO_HOME', ARDUINO_HOME_DEFAULT)
ARDUINO_PORT    = ARGUMENTS.get('ARDUINO_PORT', ARDUINO_PORT_DEFAULT)
ARDUINO_BOARD   = ARGUMENTS.get('ARDUINO_BOARD', ARDUINO_BOARD_DEFAULT)
ARDUINO_VER     = ARGUMENTS.get('ARDUINO_VER', 0) # Default to 0 if nothing is specified
RST_TRIGGER     = ARGUMENTS.get('RST_TRIGGER', None) # use built-in pulseDTR() by default
EXTRA_LIB       = ARGUMENTS.get('EXTRA_LIB', None) # handy for adding another arduino-lib dir
SKETCHBOOK_HOME = ARGUMENTS.get('SKETCHBOOK_HOME', SKETCHBOOK_HOME_DEFAULT) # If set will add the libraries dir from the sketchbook

if not ARDUINO_HOME:
    print 'ARDUINO_HOME must be defined.'
    raise KeyError('ARDUINO_HOME')

ARDUINO_CORE = pathJoin(ARDUINO_HOME, 'hardware/arduino/cores/arduino')
ARDUINO_SKEL = pathJoin(ARDUINO_CORE, 'main.cpp')
ARDUINO_CONF = pathJoin(ARDUINO_HOME, 'hardware/arduino/boards.txt')

arduino_file_path = pathJoin(ARDUINO_CORE, 'Arduino.h')
if ARDUINO_VER == 0:
    print "No Arduino version specified. Discovered version",
    if os.path.exists(arduino_file_path):
        print "100 or above"
        ARDUINO_VER = 100
    else:
        print "0023 or below"
        ARDUINO_VER = 23
else:
    print "Arduino version " + ARDUINO_VER + " specified"

# Some OSs need bundle with IDE tool-chain
if platform == 'darwin' or platform == 'win32':
    AVR_BIN_PREFIX = pathJoin(ARDUINO_HOME, 'hardware/tools/avr/bin', 'avr-')
    AVRDUDE_CONF = pathJoin(ARDUINO_HOME, 'hardware/tools/avr/etc/avrdude.conf')

ARDUINO_LIBS = [pathJoin(ARDUINO_HOME, 'libraries')]
if EXTRA_LIB:
    ARDUINO_LIBS.append(EXTRA_LIB)
if SKETCHBOOK_HOME:
    ARDUINO_LIBS.append(pathJoin(SKETCHBOOK_HOME, 'libraries'))

# check given board name, ARDUINO_BOARD is valid one
ptnBoard = re.compile(r'^(.*)\.name=(.*)')
boards = {}
for line in open(ARDUINO_CONF):
    result = ptnBoard.match(line)
    if result:
        boards[result.group(1)] = result.group(2)
if ARDUINO_BOARD not in boards:
    print "ERROR! the given board name, %s is not in the supported board list:" % ARDUINO_BOARD
    print "all available board names are:"
    for name, description in boards.iteritems():
        print "\t%s for %s" % (name.ljust(14), description)
    print "however, you may edit %s to add a new board." % ARDUINO_CONF
    sys.exit(-1)


def getBoardConf(conf):
    for line in open(ARDUINO_CONF):
        line = line.strip()
        if '=' in line:
            key, value = line.split('=')
            if key == '.'.join([ARDUINO_BOARD, conf]):
                return value
    print "ERROR! can't find %s in %s" % (conf, ARDUINO_CONF)
    assert(False)

MCU = ARGUMENTS.get('MCU', getBoardConf('build.mcu'))
F_CPU = ARGUMENTS.get('F_CPU', getBoardConf('build.f_cpu'))

# There should be a file with the same name as the folder and
# with the extension .pde or .ino
FILE_EXTENSION = ".pde" if ARDUINO_VER < 100 else ".ino"
TARGET = os.path.basename(os.path.realpath(os.curdir))
assert(os.path.exists(TARGET + FILE_EXTENSION))

cFlags = ['-ffunction-sections', '-fdata-sections', '-fno-exceptions',
          '-funsigned-char', '-funsigned-bitfields', '-fpack-struct',
          '-fshort-enums', '-Os', '-mmcu=%s' % MCU]
envArduino = Environment(CC = AVR_BIN_PREFIX + 'gcc',
                         CXX = AVR_BIN_PREFIX + 'g++',
                         AS = AVR_BIN_PREFIX + 'gcc',
                         CPPPATH = ['build/core'],
                         CPPDEFINES = {'F_CPU': F_CPU, 'ARDUINO': ARDUINO_VER},
                         CFLAGS = cFlags + ['-std=gnu99'],
                         CCFLAGS = cFlags,
                         ASFLAGS = ['-assembler-with-cpp','-mmcu=%s' % MCU],
                         TOOLS = ['gcc','g++', 'as'])

if ARDUINO_VER >= 100:
    if ARDUINO_BOARD == 'nano328':
        var = 'eightanaloginputs'
    elif ARDUINO_BOARD == 'leonardo':
        var = 'leonardo'
    elif ARDUINO_BOARD == 'mega2560':
        var = 'mega'
    elif ARDUINO_BOARD == 'micro':
        var = 'micro'
    else:
        var = 'standard'

    hwVarPath =  pathJoin(ARDUINO_HOME, 'hardware/arduino/variants', var)
    envArduino.Append(CPPPATH = hwVarPath)

def run(cmd):
    """Run a command and decipher the return code. Exit by default."""
    print ' '.join(cmd)
    try:
        check_call(cmd)
    except CalledProcessError as cpe:
        print "Error: return code: " + str(cpe.returncode)
        sys.exit(cpe.returncode)


def fnProcessing(target, source, env):
    wp = open(str(target[0]), 'wb')
    wp.write(open(ARDUINO_SKEL).read())

    types='''void
             int char word long
             float double byte long
             boolean
             uint8_t uint16_t uint32_t
             int8_t int16_t int32_t'''
    types=' | '.join(types.split())
    re_signature = re.compile(r"""^\s* (
        (?: (%s) \s+ )?
        \w+ \s*
        \( \s* ((%s) \s+ \*? \w+ (?:\s*,\s*)? )* \)
        ) \s* {? \s* $""" % (types, types), re.MULTILINE | re.VERBOSE)

    prototypes = {}

    for file in glob(os.path.realpath(os.curdir) + "/*" + FILE_EXTENSION):
        for line in open(file):
            result = re_signature.search(line)
            if result:
                prototypes[result.group(1)] = result.group(2)

    for name in prototypes.iterkeys():
        print "%s;" % name
        wp.write("%s;\n" % name)

    for file in glob(os.path.realpath(os.curdir) + "/*" + FILE_EXTENSION):
        print file, TARGET
        if not os.path.samefile(file, TARGET + FILE_EXTENSION):
            wp.write('#line 1 "%s"\r\n' % file)
            wp.write(open(file).read())

    # Add this preprocessor directive to localize the errors.
    sourcePath = str(source[0]).replace('\\', '\\\\')
    wp.write('#line 1 "%s"\r\n' % sourcePath)
    wp.write(open(str(source[0])).read())

def fnCompressCore(target, source, env):
    core_files = (x for x in imap(str, source) if x.startswith('build/core/'))
    for file in core_files:
        run([AVR_BIN_PREFIX + 'ar', 'rcs', str(target[0]), file])

bldProcessing = Builder(action = fnProcessing) #, suffix = '.cpp', src_suffix = FILE_EXTENSION)
bldCompressCore = Builder(action = fnCompressCore)
bldELF = Builder(action = AVR_BIN_PREFIX + 'gcc -mmcu=%s ' % MCU +
                          '-Os -Wl,--gc-sections -lm -o $TARGET $SOURCES')
bldHEX = Builder(action = AVR_BIN_PREFIX + 'objcopy -O ihex -R .eeprom $SOURCES $TARGET')

envArduino.Append(BUILDERS = {'Processing' : bldProcessing})
envArduino.Append(BUILDERS = {'CompressCore': bldCompressCore})
envArduino.Append(BUILDERS = {'Elf' : bldELF})
envArduino.Append(BUILDERS = {'Hex' : bldHEX})

ptnSource = re.compile(r'\.(?:c(?:pp)?|S)$')
def gatherSources(srcpath):
    return [pathJoin(srcpath, f) for f
            in os.listdir(srcpath) if ptnSource.search(f)]

# add arduino core sources
VariantDir('build/core', ARDUINO_CORE)
core_sources = gatherSources(ARDUINO_CORE)
core_sources = [x.replace(ARDUINO_CORE, 'build/core/') for x
                in core_sources if os.path.basename(x) != 'main.cpp']

# add libraries
libCandidates = []
ptnLib = re.compile(r'^[ ]*#[ ]*include [<"](.*)\.h[>"]')
for line in open(TARGET + FILE_EXTENSION):
    result = ptnLib.search(line)
    if not result:
        continue
    # Look for the library directory that contains the header.
    filename = result.group(1) + '.h'
    for libdir in ARDUINO_LIBS:
        for root, dirs, files in os.walk(libdir, followlinks=True):
            if filename in files:
                libCandidates.append(os.path.basename(root))

# Hack. In version 20 of the Arduino IDE, the Ethernet library depends
# implicitly on the SPI library.
if ARDUINO_VER >= 20 and 'Ethernet' in libCandidates:
    libCandidates.append('SPI')

all_libs_sources = []
for index, orig_lib_dir in enumerate(ARDUINO_LIBS):
    lib_dir = 'build/lib_%02d' % index
    VariantDir(lib_dir, orig_lib_dir)
    for libPath in ifilter(os.path.isdir, glob(pathJoin(orig_lib_dir, '*'))):
        libName = os.path.basename(libPath)
        if not libName in libCandidates:
            continue
        envArduino.Append(CPPPATH = libPath.replace(orig_lib_dir, lib_dir))
        lib_sources = gatherSources(libPath)
        utilDir = pathJoin(libPath, 'utility')
        if os.path.exists(utilDir) and os.path.isdir(utilDir):
            lib_sources += gatherSources(utilDir)
            envArduino.Append(CPPPATH = utilDir.replace(orig_lib_dir, lib_dir))
        lib_sources = (x.replace(orig_lib_dir, lib_dir) for x in lib_sources)
        all_libs_sources.extend(lib_sources)

# Add raw sources which live in sketch dir.
build_top = os.path.realpath('.')
VariantDir('build/local/', build_top)
local_sources = gatherSources(build_top)
local_sources = [x.replace(build_top, 'build/local/') for x in local_sources]
if local_sources:
    envArduino.Append(CPPPATH = 'build/local')

# Convert sketch(.pde) to cpp
envArduino.Processing('build/' + TARGET + '.cpp', 'build/' + TARGET + FILE_EXTENSION)
VariantDir('build', '.')

sources = ['build/' + TARGET + '.cpp']
#sources += core_sources
sources += local_sources
sources += all_libs_sources

# Finally Build!!
core_objs = envArduino.Object(core_sources)
objs = envArduino.Object(sources) #, LIBS=libs, LIBPATH='.')
objs = objs + envArduino.CompressCore('build/core.a', core_objs)
envArduino.Elf(TARGET + '.elf', objs)
envArduino.Hex(TARGET + '.hex', TARGET + '.elf')

# Print Size
# TODO: check binary size
MAX_SIZE = getBoardConf('upload.maximum_size')
print "maximum size for hex file: %s bytes" % MAX_SIZE
envArduino.Command(None, TARGET + '.hex', AVR_BIN_PREFIX + 'size --target=ihex $SOURCE')

# Reset
def pulseDTR(target, source, env):
    import serial
    import time
    ser = serial.Serial(ARDUINO_PORT)
    ser.setDTR(1)
    time.sleep(0.5)
    ser.setDTR(0)
    ser.close()

if RST_TRIGGER:
    reset_cmd = '%s %s' % (RST_TRIGGER, ARDUINO_PORT)
else:
    reset_cmd = pulseDTR

# Upload
UPLOAD_PROTOCOL = getBoardConf('upload.protocol')
UPLOAD_SPEED = getBoardConf('upload.speed')

if UPLOAD_PROTOCOL == 'stk500':
    UPLOAD_PROTOCOL = 'stk500v1'


avrdudeOpts = ['-V', '-F', '-c %s' % UPLOAD_PROTOCOL, '-b %s' % UPLOAD_SPEED,
               '-p %s' % MCU, '-P %s' % ARDUINO_PORT, '-U flash:w:$SOURCES']
if AVRDUDE_CONF:
    avrdudeOpts.append('-C %s' % AVRDUDE_CONF)

fuse_cmd = '%s %s' % (pathJoin(os.path.dirname(AVR_BIN_PREFIX), 'avrdude'),
                     ' '.join(avrdudeOpts))

upload = envArduino.Alias('upload', TARGET + '.hex', [reset_cmd, fuse_cmd])
AlwaysBuild(upload)

# Clean build directory
envArduino.Clean('all', 'build/')

# vim: et sw=4 fenc=utf-8:
