# How-To-Write-An-LLVM-Register-Allocator
This repository contains a tutorial for a quick start in how to write a register allocator using LLVM

This tutorial shows how to write a register allocator using the LLVM infrastructure, extending the `RegAllocBase` interface.

The tutorial have been writen using reStructuredText with the `sphinx` tool, the recommendations of the LLVM project have been followed (http://www.llvm.org/docs/SphinxQuickstartTemplate.html).

This content is based in my understanding of the LLVM infrastructure, which I have learned while I was working on my undergraduate thesis on the institute IFMG - campus Formiga - MG, Brazil; so if you have any suggestion, please let me know.

## Build

You can build with a variated number of output formats, please refer to the `sphinx` documentation (http://www.sphinx-doc.org/en/stable/contents.html) for instructions of installation and setup.

An example of building for PDF outuput:

    make latexpdf
