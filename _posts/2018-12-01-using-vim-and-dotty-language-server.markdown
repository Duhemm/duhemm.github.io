---
layout: post
title:  "Using Vim and the Dotty Language Server"
date:   2018-12-01 17:48:28 +0100
---

I've been asked a few times already to share my configuration to use Vim along with the Dotty
Language Server. Given that [Dotty 0.11.0-RC1 was released][dotty-0.11] yesterday and that this
release contains many improvements to Dotty IDE, there couldn't be a better time to finally write
this down.

If you're in a hurry and know your way around Vim and Dotty, [jump to the short
instructions](#for-the-impatient).

### What is Dotty IDE?

Dotty IDE is the combination of two components:

- The Dotty Language Server, which is a tool that implements the [Language Server Protocol
  (LSP)][lsp] and is used to answer the queries of the text editor
- [`vscode-dotty`][vscode-dotty], which is a VSCode extension that teaches VSCode how to start the
  Dotty Language Server and adds supports for some other features (worksheets, for instance).

VSCode is just one of the possible clients of the Dotty Language Server, therefore nothing prevents
us from reusing it with another editor. That's exactly the philosophy behind LSP.

### Setting everything up

For the rest of this post, I'll assume that you have already installed Vim, [sbt][install-sbt] and
[Coursier][install-coursier] on your machine.

#### 1 - Setting up a Dotty project

The Dotty Language Server works only with Dotty projects. If you don't have a Dotty project already,
you can set one up with:

```sh
$ sbt new lampepfl/dotty.g8
```

Once the project is ready, you can `cd` into the newly created directory and start `sbt` to make
sure that you can compile with Dotty:

```
sbt:dotty-simple> run
(...)
[info] Packaging /Users/martin/foobar/target/scala-0.11/dotty-simple_0.11-0.1.0.jar ...                                                                                  
[info] Done packaging.
[info] Running Main
Hello world!
I was compiled by dotty :)
[success] Total time: 1 s, completed Dec 1, 2018 6:52:53 PM
```

If you have it installed and are curious, you can type `launchIDE` in sbt's console to try VSCode
out: try to hover over symbols, or type some code, and you'll get some help from the language
server. Let's get something even nicer in a real editor!

#### 2 - Installing Vim plugins

If you don't use a plugin manager for Vim, I suggest that you use [Vundle][vundle]. The
[installation instructions][vundle-installation] are simple, and so is using Vundle.

I'll assume that you use Vundle.

#### 3 - We need an LSP client for Vim

There are a few LSP clients for Vim. Each has a different set of pros and cons. I am using
[`vim-lsc`][vim-lsc], because:

 - It is lightweight
 - Almost everything works out of the box
 - It is easy to configure and hack
 - It has no dependencies
 - It works on Vim and NeoVim

The installation is simple, and so is the configuration.

For `vim-lsc` to work, you'll need to have syntax highlighting enabled in Vim. Add the following to
you `.vimrc`:

```vimscript
syntax enable
```

Then, add `vim-lsc` to your plugins in `~/.vimrc`:

```vimscript
Plugin 'natebosch/vim-lsc'
```

To enable the default key bindings of `vim-lsc`, add the following to you `.vimrc`:

```vimscript
let g:lsc_auto_map = v:true
```

By itself, the LSP client is not very helpful yet. We still need to teach it how to actually start
the Dotty Language Server when a Scala file is opened. Add the following to your `.vimrc`, 

```vimscript
let g:lsc_server_commands = {
\ 'scala': "start-dotty-ide"
\ }
```

This will tell `vim-lsc` to run `start-dotty-ide` when a file whose `filetype` is `scala` is opened
for the first time. We will create this script soon, but first, let's install the plugins that we
specified in `.vimrc`: exit Vim, restart it and type `:PluginInstall`.
    
To create the startup script, Put the following in a file called `start-dotty-ide`, and write it
somewhere on your `$PATH` (you can also put it somewhere else, but remember to give the absolute
path in your vim configuration):

```sh
#!/bin/sh
if [ -f ".dotty-ide.json" ]; then
    ARTIFACT="$(cat .dotty-ide-artifact)"
    coursier launch "$ARTIFACT" -M dotty.tools.languageserver.Main -- -stdio
fi
```

Don't forget to make it executable:

```sh
$ chmod +x start-dotty-ide
```

This script will look in the directory where you opened Vim whether a file called `.dotty-ide.json`
exists. If so, it'll look into `.dotty-ide-artifact` to find the coordinates of the Dotty Language
Server artifact. It'll then give these coordinates to Coursier, which will download and start the
Dotty Language Server.

Your `.vimrc` should look at least like this:

```vimscript
 " Install and set up Vundle
set rtp+=~/.vim/bundle/Vundle.vim
call vundle#rc()
Plugin 'VundleVim/Vundle.vim'

 " Install the LSP client
Plugin 'natebosch/vim-lsc'

 " Enable syntax highlighting
syntax enable

 " Tell the LSP client how to start the server
let g:lsc_server_commands = {
\ 'scala': "start-dotty-ide"
\ }

 " Use the default bindings
let g:lsc_auto_map = v:true
```

#### 4 - Create `.dotty-ide.json`

`.dotty-ide.json` contains a description of your build (what projects exists, what are their
dependencies, what compiler options you use, and so on), and is necessary for the Dotty Language
Server to do its job.  It can be automatically created by sbt using the task `configureIDE`:

```
sbt:dotty-simple> configureIDE
```

Example of content of `.dotty-ide.json`:

```json
[ {
  "id" : "root/compile",
  "compilerVersion" : "0.11.0-RC1",
  "compilerArguments" : [ ],
  "sourceDirectories" : [ "/Users/martin/Desktop/foobar/src/main/scala", (...) ],
  "dependencyClasspath" : [ (...) ],
  "classDirectory" : "/Users/martin/Desktop/foobar/target/scala-0.11/classes"
}, (...)
]
```

You should try to manually run `start-dotty-ide` now, so that you'll notice if something is going
wrong with your script, and to let Coursier fill its cache:

```sh
$ start-dotty-ide
(...)
https://repo1.maven.org/maven2/ch/epfl/lamp/dotty-compiler_0.11/0.11.0-RC1/dotty-compil…
  100.0% [##########] 9.3 MiB (2.1 MiB / s)
  Starting server
```

You can exit with `CTRL-C`.

#### 5 - Let's start Vim!

We're all set. Start `vim` at the root of the project created by sbt new, and navigate to
`src/main/scala/Main.scala`. Move the cursor to `println` on line 4, and press `K` (`shift` + `k`).
On top of the screen, the preview window should show you the method's signature: `(x: Any): Unit`.

Congratulations, you have a working setup of Dotty IDE with Vim!

#### 6 - Wow, what can it do?

Dotty IDE and the Dotty Language Server are still actively in development, and so is `vim-lsc`.
Nevertheless, many of the features that you'd expect from an IDE work:

Completions as you type:

![Completions as you type](/assets/images/vim-dotty/completion.gif)

Go to definition:

![Go to definition](/assets/images/vim-dotty/go-to-definition.gif)

Information on hover:

![Information on hover](/assets/images/vim-dotty/hover.gif)

Find all references:

![Find all references](/assets/images/vim-dotty/find-all-references.gif)

Rename symbol:

![Rename symbol](/assets/images/vim-dotty/rename.gif)

And more! Look at [what `vim-lsc` supports][vim-lsc-features] to get an idea of what you can do.
Dotty IDE supports all of those, except code actions.

#### Some configuration tips

You can configure how you want Vim to show you references to the symbol under the cursor, or how you
want the currently active parameter to be highlighted when you request signature help. I find it
more readable to have them underlined, so I added this to my `.vimrc`:

```vimscript
highlight lscReference cterm=underline
highlight lscCurrentParameter cterm=underline
```

I don't really like the default configuration for the completion menu, so I use:

```vimscrip
set completeopt=menu,menuone,longest
" Use <C-j/k> for going down/up in completion menu
inoremap <expr> <C-j> pumvisible() ? "\<C-n>" : "\<C-j>"
inoremap <expr> <C-k> pumvisible() ? "\<C-p>" : "\<C-k>"
```

Also, I want to be able to use `<enter>` to select the highlighted completion:
```
" Enter selects the highlighted completion when the menu is shown
imap <expr> <CR> pumvisible()
                 \ ? "\<C-Y>"
                 \ : "<Plug>delimitMateCR"
```

Note that the else branch uses `<Plug>delimitMateCR`, because I still want indentation to work
properly. If you don't use [delimitMate][delimitMate], you can replace the else branch with
`"\<C-g>u\<CR>"`.

Finally, I don't want the preview window to stick around when I use hover or signature help:
```vimscript
" Discard preview window when the cursor is moved
autocmd CursorMoved * if pumvisible() == 0|pclose|endif
autocmd InsertLeave * if pumvisible() == 0|pclose|endif
```

### For the impatient

#### 1 - Set up Vim
Minimal `.vimrc`:

```vimscript
set rtp+=~/.vim/bundle/Vundle.vim
call vundle#rc()
Plugin 'VundleVim/Vundle.vim'
Plugin 'natebosch/vim-lsc'

syntax enable

let g:lsc_server_commands = {
\ 'scala': "start-dotty-ide"
\ }

let g:lsc_auto_map = v:true
```

Exit Vim, restart it and type `:PluginInstall`.

#### 2 - Write a script to start the Dotty Language Server

Write the following script in `start-dotty-ide`, make it executable and on your `$PATH`:

```sh
#!/bin/sh
if [ -f ".dotty-ide.json" ]; then
    ARTIFACT="$(cat .dotty-ide-artifact)"
    coursier launch "$ARTIFACT" -M dotty.tools.languageserver.Main -- -stdio
fi
```

#### 3 - Create the configuration in sbt

```
sbt:your-project> configureIDE
```

#### 4 - Start Vim

That's it.

### Troubleshooting

Dotty IDE is still in active development, and you will encounter errors and bugs. You can get more
information about what's happening by keeping a log of the communication between Vim and the Dotty
Language Server. Create a new script `start-dotty-ide-debug`, with content:

```sh
#!/bin/sh
tee $HOME/log-in.txt | start-dotty-ide 2> $HOME/log-err.txt | tee $HOME/log-out.txt
```

Make it executable, and replace the command that is used to start the language server in your
`.vimrc`. The messages sent by the client to the server can be read in `$HOME/log-in.txt`, the
messages from the server to the client will be in `$HOME/log-out.txt`, and the error log of the
server will be in `$HOME/log-err.txt`.

The last messages that were exchanged between the client and the server, along with stack traces can
be invaluable when trying to reproduce bugs. Please include them in your bug reports!

The Dotty issue tracker lives on GitHub:
[https://github.com/lampepfl/dotty/issues](https://github.com/lampepfl/dotty/issues)

[dotty-0.11]: http://dotty.epfl.ch/blog/2018/11/30/11th-dotty-milestone-release.html
[delimitMate]: https://github.com/Raimondi/delimitMate
[derekwyatt/vim-scala]: https://github.com/derekwyatt/vim-scala
[install-coursier]: https://github.com/coursier/coursier/blob/42e5a7f6b3b0316b615d76762f1a3146cd3d3efb/doc/FORMER-README.md#command-line
[install-sbt]: https://www.scala-sbt.org/1.0/docs/Setup.html
[lsp]: https://microsoft.github.io/language-server-protocol
[vim-lsc]: https://github.com/natebosch/vim-lsc
[vim-lsc-features]: https://github.com/natebosch/vim-lsc#configuration
[vscode-dotty]: https://marketplace.visualstudio.com/items?itemName=lampepfl.dotty
[vundle]: https://github.com/VundleVim/Vundle.vim
[vundle-installation]: https://github.com/VundleVim/Vundle.vim#quick-start
