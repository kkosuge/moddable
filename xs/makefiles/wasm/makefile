#
# Copyright (c) 2016-2017  Moddable Tech, Inc.
#
#   This file is part of the Moddable SDK Tools.
# 
#   The Moddable SDK Tools is free software: you can redistribute it and/or modify
#   it under the terms of the GNU General Public License as published by
#   the Free Software Foundation, either version 3 of the License, or
#   (at your option) any later version.
# 
#   The Moddable SDK Tools is distributed in the hope that it will be useful,
#   but WITHOUT ANY WARRANTY; without even the implied warranty of
#   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#   GNU General Public License for more details.
# 
#   You should have received a copy of the GNU General Public License
#   along with the Moddable SDK Tools.  If not, see <http://www.gnu.org/licenses/>.
#
.NOTPARALLEL:

XS_DIR ?= $(MODDABLE)/xs
export XS_DIR
BUILD_DIR ?= $(MODDABLE)/build
export BUILD_DIR

all: debug release

debug:
	make -f xsc.mk
	make -f xscl.mk
	make -f xsl.mk

release:
	make GOAL=release -f xsc.mk
	make GOAL=release -f xscl.mk
	make GOAL=release -f xsl.mk
	
