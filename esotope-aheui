#!/usr/bin/env python
"""\
Aheui Interpreter in Python, revision 5a
Copyright (c) 2005, Kang Seonghoon (Tokigun).

This library is free software; you can redistribute it and/or
modify it under the terms of the GNU Lesser General Public
License as published by the Free Software Foundation; either
version 2.1 of the License, or (at your option) any later version.

This library is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
Lesser General Public License for more details.

You should have received a copy of the GNU Lesser General Public
License along with this library; if not, write to the Free Software
Foundation, Inc., 51 Franklin St, Fifth Floor, Boston, MA  02110-1301  USA

$Id: aheui.py,v 1.3 2005/07/28 11:00:31 tokigun Exp $
"""

import sys

class AheuiStack(list):
    def push(self, value):
        self.append(value)

    def pop(self):
        if len(self): return list.pop(self)
        return None

    def dup(self):
        if len(self): self.append(self[-1])

    def swap(self):
        if len(self) > 1:
            self[-1], self[-2] = self[-2], self[-1]

class AheuiQueue(AheuiStack):
    def pop(self):
        if len(self): return list.pop(self, 0)
        return None

class AheuiPort(AheuiStack):
    def push(self, value):
        pass

    def pop(self):
        return None

class AheuiIO:
    def __init__(self, encoding='utf-8'):
        self.encoding = encoding

    def getunicode(self):
        buf = ''
        while 1:
            char = sys.stdin.read(1)
            if char == '': return -1
            buf += char
            try:
                return ord(buf.decode(self.encoding))
            except UnicodeError:
                try:
                    reason = sys.exc_info()[1].reason
                    if not ('end of data' in reason or 'incomplete' in reason):
                        return None
                except: pass

    def getunichar(self):
        char = self.getunicode()
        if char is None:
            return u'\ufffd' # U+FFFD: Replacement Character
        elif char < 0:
            return u''
        else:
            return unichr(char)

    def getchar(self):
        char = self.getunicode()
        if char is None:
            return 0xfffd
        else:
            return char == 13 and 10 or char

    def getint(self):
        str = u' '; char = u'0'; sign = 1
        while str and not str.isdigit():
            if str == u'-':
                sign = -sign
            elif str != u'+':
                sign = 1
            str = self.getunichar()
        while char.isdigit():
            char = self.getunichar()
            str += char
        return sign * int(str[:-1])
    
    def putint(self, value):
        sys.stdout.write(unicode(value).encode(self.encoding))

    def putchar(self, value):
        try: sys.stdout.write(unichr(value).encode(self.encoding))
        except: sys.stdout.write('[U+%04X]' % value)

class AheuiCode:
    def __init__(self, code):
        if not isinstance(code, unicode):
            code = str(code).encode('utf-8')
        
        self.space = []
        self.sizex = self.sizey = 0
        for line in code.splitlines():
            if len(line) > self.sizex:
                self.sizex = len(line)
            self.sizey += 1
            self.space.append([self.split(char) for char in line])
        if self.sizex == 0: self.sizex = 1
        if self.sizey == 0: self.sizey = 1

    def __getitem__(self, index):
        try:
            return self.space[index[1]][index[0]]
        except IndexError:
            return ()

    def split(self, char):
        if u'\uac00' <= char <= u'\ud7a3':
            code = ord(char) - 0xac00
            return (code/588, (code/28)%21, code%28)
        else:
            return ()

    split = classmethod(split)

################################################################################

class AheuiInterpreter:
    nrequired = [0, 0, 2, 2, 2, 2, 1, 0, 1, 0, 1, 0, 2, 0, 1, 0, 2, 2, 0]
    finalvalue = [0, 2, 4, 4, 2, 5, 5, 3, 5, 7, 9, 9, 7, 9, 9,
                  8, 4, 4, 6, 2, 4, 1, 3, 4, 3, 4, 4, 3]

    def __init__(self, code, io=None):
        self.code = code
        self.io = io
        self.ready = False

    def init(self):
        self.x = 0; self.y = 0
        self.dx = 1; self.dy = 0
        self.stack = 0
        self.ready = True
        self.finished = False
        
        self.stacks = []
        for i in xrange(28):
            if i == 21:
                self.stacks.append(AheuiQueue())
            elif i == 27:
                self.stacks.append(AheuiPort())
            else:
                self.stacks.append(AheuiStack())
        self.current = self.stacks[self.stack]

    def get_delta(self, code, dx, dy):
        if code == 0: dx, dy = 1, 0
        elif code == 2: dx, dy = 2, 0
        elif code == 4: dx, dy = -1, 0
        elif code == 6: dx, dy = -2, 0
        elif code == 8: dx, dy = 0, -1
        elif code == 12: dx, dy = 0, -2
        elif code == 13: dx, dy = 0, 1
        elif code == 17: dx, dy = 0, 2
        elif code == 18: dy = -dy
        elif code == 19: dx, dy = -dx, -dy
        elif code == 20: dx = -dx
        return (dx, dy)

    def next_cell(self, x, y, dx, dy, sizex, sizey):
        x += dx; y += dy
        if x >= sizex: x = x % dx
        elif x < 0: x = sizex - (sizex - x) % dx
        if y >= sizey: y = y % dy
        elif y < 0: y = sizey - (sizey - y) % dy

        return (x, y)

    def step(self):
        char = self.code[self.x, self.y]

        if char != ():
            a, b, c = char
            self.dx, self.dy = self.get_delta(b, self.dx, self.dy)

            if len(self.current) < self.nrequired[a]:
                self.dx = -self.dx; self.dy = -self.dy
            elif a == 2:
                x = self.current.pop()
                y = self.current.pop()
                self.current.push(x != 0 and y / x or 0)
            elif a == 3:
                x = self.current.pop()
                y = self.current.pop()
                self.current.push(y + x)
            elif a == 4:
                x = self.current.pop()
                y = self.current.pop()
                self.current.push(y * x)
            elif a == 5:
                x = self.current.pop()
                y = self.current.pop()
                self.current.push(x != 0 and y % x or 0)
            elif a == 6:
                x = self.current.pop()
                if c == 21:
                    self.io.putint(x)
                elif c == 27:
                    self.io.putchar(x)
            elif a == 7:
                if c == 21:
                    x = self.io.getint()
                elif c == 27:
                    x = self.io.getchar()
                else:
                    x = self.finalvalue[c]
                self.current.push(x)
            elif a == 8:
                self.current.dup()
            elif a == 9:
                self.stack = c
                self.current = self.stacks[c]
            elif a == 10:
                x = self.current.pop()
                self.stacks[c].push(x)
            elif a == 12:
                x = self.current.pop()
                y = self.current.pop()
                self.current.push(y>=x and 1 or 0)
            elif a == 14:
                if self.current.pop() == 0:
                    self.dx = -self.dx; self.dy = -self.dy
            elif a == 16:
                x = self.current.pop()
                y = self.current.pop()
                self.current.push(y - x)
            elif a == 17:
                self.current.swap()
            elif a == 18:
                self.finished = True
        
        self.x, self.y = self.next_cell(self.x, self.y, self.dx, self.dy, self.code.sizex, self.code.sizey)

    def execute(self):
        self.init()
        while not self.finished:
            self.step()

    get_delta = classmethod(get_delta)
    next_cell = classmethod(next_cell)

################################################################################

OPT_SKIP_BLANKLINE = 1

def version(apppath):
    print __doc__
    return 0

def help(apppath):
    print __doc__[:__doc__.find('\n\n')]
    print
    print "Usage: %s [options] <filename> [<param>]" % apppath
    print
    print "--help, -h"
    print "    prints help message."
    print "--version, -V"
    print "    prints version information."
    print "--skip-blankline, -x"
    print "    skips leading blank lines."
    print "--fileencoding, -e"
    print "    set encoding of source code. (default 'utf-8')"
    print "--ioencoding"
    print "    set encoding for input/output."
    return 0

def main(argv):
    import getopt, locale

    try:
        opts, args = getopt.getopt(argv[1:], "hVe:x",
                ['help', 'version', 'fileencoding=', 'ioencoding=', 'charset=',
                 'skip-blankline'])
    except getopt.GetoptError:
        return help(argv[0])

    fencoding = 'utf-8'
    ioencoding = locale.getdefaultlocale()[1]
    mode = 0
    for opt, arg in opts:
        if opt in ('-h', '--help'):
            return help(argv[0])
        if opt in ('-V', '--version'):
            return version(argv[0])
        if opt in ('-e', '--fileencoding', '--charset'):
            if opt == '--charset':
                print 'Warning: --charset option is deprecated. use --fileencoding.'
            fencoding = arg
        if opt in ('-x', '--skip-blankline'):
            mode |= OPT_SKIP_BLANKLINE
        if opt == '--ioencoding':
            ioencoding = arg
    if len(args) == 0:
        print __doc__[:__doc__.find('\n\n')]
        return 1

    # encoding fix
    if fencoding == 'euc-kr': fencoding = 'cp949'
    if ioencoding == 'euc-kr': ioencoding = 'cp949'

    filename = args[0]
    param = args[1:]
    try:
        if filename == '-':
            data = sys.stdin.read()
        else:
            data = file(filename).read()
    except IOError:
        print "Cannot read file: %s" % filename
        return 1
    try:
        data = data.decode(fencoding)
    except LookupError:
        print "There is no encoding named '%s'." % fencoding
        return 1
    except UnicodeDecodeError:
        print "Unicode decode error!"
        return 1

    if mode & OPT_SKIP_BLANKLINE:
        data = data.lstrip('\r\n')
    code = AheuiCode(data)
    io = AheuiIO(ioencoding)
    interpreter = AheuiInterpreter(code, io)
    interpreter.execute()
    return interpreter.current.pop() or 0

if __name__ == '__main__':
    sys.exit(main(sys.argv))

