#!/usr/bin/python

# scons script for the Arduino sketch
#
# Copyright (C) 2010 by Homin Lee <ff4500@gmail.com>
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; either version 2 of the License, or
# (at your option) any later version.

# Basic Usage:
# 1. make a folder which have same name of the sketch (ex. Blink/ for Blik.pde)
# 2. put the sketch, SConstruct(this file) and pulsedtr.py under the folder.
# 3. to make the HEX. do following in the folder.
#     $ scons
# 4. to upload the binary, do following in the folder.
#     $ scons upload

# Change Configs:
# by default, this script configured for;
# - "/usr/share/arduino/" as Arduino home.
# - "atmega168" base Arduino.
# - "/deb/ttyUSB0" as upload port.
# - "pulsedtr.py" as reset trigger.
# and can modify the configs via command-line options.
# to find out available options, find "ARGUMETS.get(..)"s in following souce.

# Ex. Run with custom configuration.
#     $ scons mcu=atmega8
#     $ scons port=/dev/ttyS0 upload

from glob import glob
import sys
import re
import os
pathJoin = os.path.join

ARDUINO_HOME	= ARGUMENTS.get('arduino', '/usr/share/arduino/') #'/apps/arduino-0018/'
UPLOAD_PORT	= ARGUMENTS.get('port', '/dev/ttyUSB0')
BOARD		= ARGUMENTS.get('board', 'atmega328')
#RST_TRIGGER	= ARGUMENTS.get('rst', './pulsedtr.py')
RST_TRIGGER	= ARGUMENTS.get('rst', '')

# TODO: find arduino ver dynamically
ARDUINO_VER	= 19 # Arduino 0019
AVR_PREFIX	= 'avr-'

ARDUINO_CORE	= ARDUINO_HOME+'hardware/arduino/cores/arduino/'
ARDUINO_SKEL	= ARDUINO_CORE+'main.cpp'
ARDUINO_CONF	= ARDUINO_HOME+'hardware/arduino/boards.txt'
ARDUINO_LIBS	= ARDUINO_HOME+'libraries/'
# Can add more library-directory in another -not under ARDUINO_HOME- path.
#ARDUINO_LIBS	+= '/apps/arduino_libs/'

# check given board name, BOARD is valid one
ptnBoard = re.compile(r'^(.*)\.name=(.*)')
boards = {}
for line in open(ARDUINO_CONF):
    result = ptnBoard.findall(line)
    if result:
        boards[result[0][0]] = result[0][1]
if not BOARD in boards.keys():
    print ("ERR! given boardname, %s is not in supported board list."%BOARD)
    print ("all available boardname is following:")
    for name in boards.keys():
        print ("\t%s\t for %s"%(name, boards[name]))
    print ("maybe, you want edit %s for add new board."%ARDUINO_CONF)
    sys.exit(-1)

def getBoardConf(strPtn):
    ptn = re.compile(strPtn)
    for line in open(ARDUINO_CONF):
        result = ptn.findall(line)
        if result:
            return result[0]
    assert(False)

MCU = getBoardConf(r'^%s\.build\.mcu=(.*)'%BOARD)
F_CPU = getBoardConf(r'^%s\.build\.f_cpu=(.*)'%BOARD)

# There should be a file with the same name as the folder and with the extension .pde
TARGET = os.path.basename(os.path.realpath(os.curdir))

cFlags = ['-ffunction-sections', '-fdata-sections', '-fno-exceptions',
    '-funsigned-char', '-funsigned-bitfields', '-fpack-struct', '-fshort-enums',
    '-Os', '-mmcu=%s'%MCU]
envArduino = Environment(CC = AVR_PREFIX+'gcc',
    CXX = AVR_PREFIX+'g++',
    CPPPATH = ['core'],
    CPPDEFINES = {'F_CPU':F_CPU, 'ARDUINO':ARDUINO_VER},
    CFLAGS = cFlags+['-std=gnu99'],
    CCFLAGS = cFlags
    )

def fnProcessing(target, source, env):
    wp = open ('%s'%target[0], 'w')
    #wp.write('#include "WProgram.h"\n')
    wp.write(open(ARDUINO_SKEL).read())
    wp.write(open('%s'%source[0]).read())
    wp.close()
    return None

envArduino.Append(BUILDERS = {'Processing':Builder(action = fnProcessing,
        suffix = '.cpp', src_suffix = '.pde')})
envArduino.Append(BUILDERS={'Elf':Builder(action=AVR_PREFIX+'gcc '+
        '-mmcu=%s -Os -Wl,--gc-sections -o $TARGET $SOURCES -lm'%MCU)})
envArduino.Append(BUILDERS={'Hex':Builder(action=AVR_PREFIX+'objcopy '+
        '-O ihex -R .eeprom $SOURCES $TARGET')})

# add arduino core sources
VariantDir('core', ARDUINO_CORE)
gatherSources = lambda x: glob(pathJoin(x, '*.c'))+\
        glob(pathJoin(x, '*.cpp'))+\
        glob(pathJoin(x, '*.S'))
core_sources = gatherSources(ARDUINO_CORE)
core_sources = filter(lambda x: not (os.path.basename(x) == 'main.cpp'), core_sources)
core_sources = map(lambda x: x.replace(ARDUINO_CORE, 'core/'), core_sources)

# add libraries
libCandidates = []
ptnLib = re.compile(r'#include <(.*)\.h>')
for line in open (TARGET+'.pde'):
    result = ptnLib.findall(line)
    if result:
        libCandidates += result

VariantDir('ext_libs', ARDUINO_LIBS)
lib_sources = []
for libPath in filter(os.path.isdir, glob(ARDUINO_LIBS + '*')):
    libName = os.path.basename(libPath)
    if not libName in libCandidates:
        continue
    envArduino.Append(CPPPATH = libPath.replace(ARDUINO_LIBS, 'ext_libs/'))
    lib_sources = gatherSources(libPath)
    utilDir = pathJoin(libPath, 'utility')
    if os.path.exists(utilDir) and os.path.isdir(utilDir):
        lib_sources = gatherSources(utilDir)
        envArduino.Append(CPPPATH = utilDir.replace(ARDUINO_LIBS, 'ext_libs'))
    lib_sources = map(lambda x: x.replace(ARDUINO_LIBS, 'ext_libs/'), lib_sources)

sources = ['build/'+TARGET+'.cpp']
sources += lib_sources
sources += core_sources

# Convert sketch(.pde) to cpp
envArduino.Processing('build/'+TARGET+'.cpp', 'build/'+TARGET+'.pde')
VariantDir('build', '.')

# Finally Build!!
objs = envArduino.Object(sources) #, LIBS=libs, LIBPATH='.')
envArduino.Elf(TARGET+'.elf', objs)
envArduino.Hex(TARGET+'.hex', TARGET+'.elf')

# Print Size
# TODO: check binary size
MAX_SIZE = getBoardConf(r'^%s\.upload.maximum_size=(.*)'%BOARD)

envArduino.Command(None, TARGET+'.hex', 'avr-size --target=ihex $SOURCE')

# Reset
def pulseDTR(target, source, env):
    import serial
    import time
    ser = serial.Serial(UPLOAD_PORT)
    ser.setDTR(1)
    time.sleep(0.5)
    ser.setDTR(0)
    ser.close()

if RST_TRIGGER:
    reset_cmd = '%s %s'%(RST_TRIGGER, UPLOAD_PORT)
else:
    reset_cmd = pulseDTR

# Upload 
UPLOAD_PROTOCOL = getBoardConf(r'^%s\.upload\.protocol=(.*)'%BOARD)
UPLOAD_SPEED = getBoardConf(r'^%s\.upload\.speed=(.*)'%BOARD)

avrdudeOpts = ['-V', '-F', '-c %s'%UPLOAD_PROTOCOL, '-b %s'%UPLOAD_SPEED,
    '-p %s'%MCU, '-P %s'%UPLOAD_PORT, '-U flash:w:$SOURCES']
fuse_cmd = 'avrdude %s'%(' '.join(avrdudeOpts))

upload = envArduino.Alias('upload', TARGET+'.hex', [reset_cmd, fuse_cmd]);
AlwaysBuild(upload)

# vim: et sw=4 fenc=utf-8:
