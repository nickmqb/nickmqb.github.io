---
layout: post
title:  Building a language server for Muon
date:   2019-11-24 23:00:00 +0100
---

I'm a big fan of interactive tool support for programming. I often use features like "go to definition", "find symbol", compile error underlining, etc. Also, earlier this year, I launched [Muon, a new low-level programming language](https://github.com/nickmqb/muon). As no tooling existed beyond the compiler, writing the initial Muon programs (such as a [self hosted compiler](https://github.com/nickmqb/muon/tree/master/compiler)) turned out to be a pretty interesting experience -- for the first time in years I found myself writing code using nothing more than a text editor.

While I actually missed my tools less than I thought I would, I did find that the lack of tool support slowed me down in some places. Statically typed languages in particular lend themselves well to a high level of tool support, and as Muon is a statically typed language, the contrast with various tools for mature languages like C and C# was pretty stark. So, I figured it was time to remedy this situation.

## Language server protocol

[The language server protocol (LSP)](https://microsoft.github.io/language-server-protocol/) is an open source project that aims to standardize various interfaces for language tool support. Rather than writing custom code for many editors, a language author just needs to implement the LSP once, by writing a so-called language server. [Quite a few editors](https://microsoft.github.io/language-server-protocol/implementors/tools/) have already implemented language clients for the protocol, which is pretty nice, because it allows people to use their favorite editor with any supported language.

[The protocol](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/) is text based; messages are encoded as JSON objects. The editor (the client) invokes the server (the language server) as needed, telling it about various events that have occurred in the editor, and the language server can respond appropriately. The protocol does not specify a transport mechanism, but at a glance, exchanging messages over standard input/output seems to be supported for the editors that I looked into. Seems reasonable enough, so let's try it out!

## Minimal feature set

The LSP is quite extensive. It defines a whole range of operations that a language server may want to handle. The protocol allows for both clients (editors) and servers to implement a subset of the available feature set, and provides a way to negotiate this subset. For now, I've chosen to just support three features in the Muon language server:

* Find symbol by name
* Go to definition
* As-you-type diagnostics (i.e. "live error underlining")

This seemed like a good starting point for trying out the protocol in practice. I picked VS Code as my editor; as it's built by Microsoft (like the LSP itself) I figured it would probably provide the most canonical implementation of the protocol. Moreover, I reasoned that it would probably support the protocol out of the box. The latter actually turned out to not be the case: we need to create a new VS Code extension, pull in the [vscode-language-client](https://www.npmjs.com/package/vscode-languageclient) npm module, and finally write some glue code and configs. Fortunately, the steps are [well documented](https://code.visualstudio.com/api/language-extensions/language-server-extension-guide), so everything is pretty quick to set up. As a bonus, the extension provides a way to include a grammar for syntax highlighting, which can be handled separately from the LSP.

## Protocol messages

The LSP distinguishes between requests, which are messages that must be responded to by the server, and notifications, which are messages that do not require a response.

* The client (editor) starts the language server and sends the `initialize` request. This is where feature negotiation occurs (the client and server essentially exchange a list of supported feature flags), and the request also informs us about various settings, such as the editor workspace root path.

* In the initial handshake we can ask the client to keep us up-to-date on opened documents (via `textDocument/didOpen` notifications) and changes to them (via `textDocument/didChange` notifications). By monitoring both we can maintain a copy of the state of each editor document. Note that we cannot just read the files from disk ourselves, as the user may have made changes that they have not yet saved to disk.

* The client sends the `workspace/symbol` request when the user searches for a symbol, and sends the `textDocument/definition` request when the user invokes "go to definition" while the cursor is on a symbol in the document. We (the server) can send the client a `textDocument/publishDiagnostics` notification to tell the client about any errors that we've found.

## Writing the language server

That's all we need to know to write a basic server! Of course, we're going to write it in Muon :). Our design is pretty straightforward: the server runs a message loop that reads server messages from stdin and responds to the ones that we care about over stdout. As stated above, we maintain a copy of the state of each editor document by monitoring `didOpen` and `didChange` notifications.

On every change, we run the compiler. **Running the compiler on every key stroke may seem a bit excessive, but scales better than one might expect!** We can run straight from memory (no disk I/O needed) and we just need to run the parse and type check stages -- code generation stages can be omitted. For example, let's suppose we're working on the Muon compiler itself. The project consists of ~12K lines of code. Parsing and type checking, together, take just ~12ms (on my machine). That's less than a frame of latency on an average system! Of course, this number will go up with the size of a code base. Some form of incremental compilation would still be very valuable.

The compiler returns an abstract syntax tree (AST) and a list of errors, which is all the information we need to implement support for the features that we want. When we receive a `workspace/symbol` or `textDocument/definition` request, we consult the AST to find what we need, and return a response. And, after every compilation run, we send a `publishDiagnostics` notification to the client if we've detected any errors.

## Using the language server

After putting it all together, things seem to work pretty well! Here's the VS Code extension in action:

Symbol search  
![alt text](/assets/symbol-search.gif "Symbol search")

Go to definition  
![alt text](/assets/go-to-definition.gif "Go to definition")

Error feedback  
![alt text](/assets/error-feedback.gif "Error feedback")

## Trying out other editors

Finally, it's time to see if the LSP lives up to its promise: can we use our language server from another editor? Let's try Vim, via [vim-lsc](https://github.com/natebosch/vim-lsc). After fixing a few bugs in the language server (always a good idea to test against multiple implementations of a protocol) things look pretty good:

Symbol search  
![alt text](/assets/vim-symbol-search.gif "Symbol search")

[View more GIFs here (go to definition, error feedback)](https://github.com/nickmqb/muon/blob/master/language_server/README.md#vim)

## Conclusion

If you'd like to give the language server a try for yourself, check out the [VS Code extension](https://github.com/nickmqb/vscode-muon) or have a look at the [language server README](https://github.com/nickmqb/muon/blob/master/language_server/README.md) and try it out with a LSP-compatible editor of your choice (Windows and Linux are supported -- I haven't tested macOS yet but things should work).

If you liked this post, [consider following me on twitter](https://twitter.com/nickmqb).
