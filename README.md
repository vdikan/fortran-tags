Fortran-tags
============

Emacs plugin for source code indexing of modern Fortran

Brief description
-----------------

Fortran-tags is a Fortran source code indexing tool with the main purpose of finding the definition of any variable or procedure. It is able to correctly locate all global and local variables and is expected to work with any Fortran 2008 conforming code with some minor exceptions as mentioned below. Fortran-tags is designed to be used with the Emacs text editor in a Linux environment. A prerequisite for using is Python 3.

Downloading
-----------

* The official versioned releases are available [here](https://github.com/raullaasner/fortran-tags/releases).
* The latest stable development code is available [here](https://github.com/raullaasner/fortran-tags/archive/master.zip).
* For importing the repository, issue `git clone https://github.com/raullaasner/fortran-tags.git`.

Installation
------------

1. Include `fortran-tags.el` in your Emacs configuration file,

   ```emacs-lisp
   (load-file "<path-to>/fortran-tags.el")
   ```

   Alternatively, if `fortran-tags.el` is in a standard location such as `~/.emacs.d/lisp`,

   ```emacs-lisp
   ;;(add-to-list 'load-path "~/.emacs.d/lisp")
   (require 'fortran-tags)
   ```

2. Include `fortran-tags.py` in your PATH. If Emacs can't find `fortran-tags.py`, try

   ```emacs-lisp
   (setenv "PATH" (concat "<dir-containing-fortran-tags.py>:" (getenv "PATH")))
   ```

Usage
-----

1. One must first generate a tags file with

   ```
   fortran-tags.py -g FILE [FILE ...]
   ```

   Default is to create a file named `FORTAGS` in the current directory but this can be changed with the option `-o`. For including all the source directories and subdirectories of the project, one can issue, for example,

   ```
   find -name '*.f90' | xargs fortran-tags.py -g
   ```

   (Why not create an alias for this?) When adding, removing, or modifying the source files only minimal changes are made to the tags file. The structure of the tags file is explained at the beginning of `fortran-tags.py`.

2. The Elisp command `fortran-find-tag` tries to find the definition of the word under the cursor. It first searches for the definition in the present scope. If not found, it expands the search scope, tries again, and if still not found, keeps expanding the scope until searching at the module level. If the definition is not found in the current module, the search expands to the module wide variables and procedures of all other modules. If more than 1 match was found, `fortran-goto-next` can be used to cycle through all the matches. If a match was found in the current module, then the other modules are not searched in because, in principle, this has to be the correct match. An exception can occur when using operator overloading (see below). If one wishes, a global search can always be forced by issuing `fortran-goto-next` even if just 1 match was found.

3. The location of the tags file is determined with the first issuance of `fortran-find-tag` or by putting

   ```emacs-lisp
   (setq fortran-tags-path "~/my-project/FORTAGS")
   ```

   into a project related configuration file. The tags file is read from the hard drive each time `fortran-find-tag` is invoked (searches are based on `grep`). Thus, when the tags file is regenerated the definitions are instantly up to date without restarting the editor.

   A simple elisp function to regenerate the tags file is

   ```emacs-lisp
   (defun generate-fortran-tags (path)
     (compile
      (concat "find " path
              " -name \"*.f90\" | xargs fortran-tags.py -o " path "/FORTAGS -g"))
     (setq fortran-tags-path (concat path "/FORTAGS")))
   ```

   where path is the root directory of a project. A call to this function could be included in a project related configuration file that is always run when a project is opened.

4. `fortran-pop-tag-mark` goes back to the previous position (works repeatedly).

5. `fortran-find-proc-calls` finds all calls to the procedure under the cursor. The results can then be cycled through using `fortran-goto-next`. This function is not perfect and can return some garbage along with the correct results. The reasons are that it
   1. also searches in strings and comments;
   2. is unable to find a match if `&` is used in highly unusal ways;
   3. is unable to distinguish between type-bound arrays and procedures.

   If the code is well written with a clean style and the user knows whether they are searching for a subroutine or a function, the search can be sped up using the following functions, which employ a simpler regex pattern and search specifically for a subroutine or a function:
   * `fortran-find-proc-calls-sub` - searches only for subroutine calls using the search pattern <code>^&nbsp;\*call&nbsp;X&nbsp;\*([(&]|$)</code>, where `X` is the word under the cursor.
   * `fortran-find-proc-calls-func` - searches only for function calls, excluding type-bound procedures, using the search pattern <code>[=+/\*(&\-]&nbsp;\*X&nbsp;\*[(&]</code>.
   * `fortran-find-proc-calls-type` - searches only for type-bound procedures using <code>%X&nbsp;\*[(&]</code>.

   These specialized searches are all case sensitive, while the general search with `fortran-find-proc-calls` is case insensitive and uses the search patterns <code>(^|[;&])&nbsp;\*call&nbsp;+X&nbsp;\*([(&;\!]|$)</code> (for subroutines) and <code>([=+/\*(%&-]|^)&nbsp;\*X&nbsp;\*[(&]</code> (for functions).

6. In order to specify the search term from keyboard, place the cursor so that it's not covering any character (`_A-Za-z0-9`), and invoke any search function, e.g., `fortran-find-tag`. All functions except `fortran-find-proc-calls` are case sensitive.

7. `fortran-procedures-in-buffer` lists all subroutines and functions in the current buffer.

8. The default key-bindings are

   ```
   M-.   fortran-find-tag
   M-*   fortran-pop-tag-mark
   M-n   fortran-goto-next
   M-s g fortran-find-proc-calls
   M-s s fortran-find-proc-calls-sub
   M-s f fortran-find-proc-calls-func
   M-s t fortran-find-proc-calls-type
   M-s d fortran-procedures-in-buffer
   ```

   These can be disabled by turning off `fortran-tags-mode`.

Limitations
-----------

* If the definition is found not in the present module but elsewhere, `fortran-find-tag` may return more than one match because it is unable to determine the exact origin of the word under the cursor.

* It is assumed that the semicolon and ampersand are not used in highly non-standard ways.

* If the initialization of a variable spans several lines, then only the first line is processed. In this example,

   ```fortran
   integer :: a(4) = [ 1, 2, &
                      & 3, 4], b
   integer :: c = 5
   ```

   `a` and `c` are correctly written into the tags file, while `b` is not found. `a` could also be a scalar with some complex initialization expression.

* Submodules are not supported.

* If there are two or more `associate` constructs within the same scope that contain an identical keyword, then the duplicate definitions are not found.

* If the result variable of a function is defined using both the `result` keyword and a dummy variable, only the former counts as a definition. It is assumed that the return type is declared on the same line as function declaration.

* If a procedure is overloaded using multiple interface blocks which are scattered over different modules, one of which is the current module, then only the definition in the current module is reached. In such a situation, one can invoke `fortran-goto-next`, which always performs a global search. If no match was found in the current module, then `fortran-find-tag` always finds all of them.

* It is recommended not to end subprograms with a bare `end` (not followed by, e.g., `subroutine`). It will often work but we can't guarantee all cases.

* Using `*` for specifying the kind is not recommended, but it will work with `real*8` and `complex*16`. See `exceptions` in fortran-tags.py. Other nonstandard syntax can be included by modifying this variable, e.g., by setting `exceptions = 'real[*]8|complex[*]16|integer*8|'`. When dealing with clean code only, one can set `exceptions = ''`.

* Fortran-tags is unable to correctly handle the `#include` and `include` directives. The generated tags file might not have the correct contents.


Troubleshooting
---------------

A good place to bring up any issues is https://github.com/raullaasner/fortran-tags/issues.

Call for developers
-------------------

Any help for porting this to other platforms is most welcome.

License
-------

This project is distributed under the terms of the GNU General Public License, see LICENSE in the root directory of the present distribution or http://gnu.org/copyleft/gpl.txt.
