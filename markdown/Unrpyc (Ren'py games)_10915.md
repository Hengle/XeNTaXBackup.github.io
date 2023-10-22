## Post #1
- Username: ikskoks
- Rank: Moderator
- Number of posts: 1671
- Joined date: Fri Jul 27, 2012 12:06 am
- Post datetime: 2013-10-30T10:30:39+00:00
- Post Title: Unrpyc (Ren'py games)

> Unrpyc is a script to decompile Ren'Py (http://www.renpy.org/) compiled .rpyc
>
> script files. It will not extract files from .rpa archives. For that, use rpatool
>
> (https://github.com/Shizmob/rpatool), or UnRPA (https://github.com/Lattyware/unrpa).
>
> 
>
> Unrpyc needs some internal Ren'Py structures to work. You can either use the -b 
>
> argument to specify the directory in which renpy lies, place the renpy module
>
> in your Python module search path, or copy the renpy folder to the unrpyc directory.
>
> 
>
> Usage options:
>
> 
>
> Options:
>
>   --version      show program's version number and exit
>
>   -h, --help     show this help message and exit
>
>   -c, --clobber  overwrites existing output files
>
>   -b, --basedir  specify the directory in which the renpy folder 
>
>                  resides. using this it is not necessary for the renpy 
>
>                  directory to be in python's search path, or in the 
>
>                  unrpyc directory.
>
> 
>
> Usage: unrpyc.py [options] script1 script2 ...
>
> 
>
> You can give several .rpyc files on the command line. Each script will be
>
> decompiled to a corresponding .rpy on the same directory. By default, the
>
> program will not overwrite existing files, use -c to do that.
>
> 
>
> This script will try to disassemble all AST nodes. In the case it encounters an unknown 
>
> node type, which may be caused by an update to Ren'Py somewhere in the future, a
>
> warning will be printed and a placeholder inserted in the script when it finds a
>
> node it doesn't know how to handle. If you encounter this, please open an issue 
>
> to alert us of the problem.  
>
> 
>
> https://github.com/yuriks/unrpyc
[unrpyc.zip](https://xentaxbackup.github.io/file/6742_unrpyc.zip)
