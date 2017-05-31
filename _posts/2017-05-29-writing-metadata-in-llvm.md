---
title: Writing Metadata in LLVM
layout: post
date: '2017-05-29 19:20:00'
author: Jiten Thakkar
tags:
- C
- LLVM
- Metadata
- Code
comments: true
---

This is a small post to demonstrate working code for writng and reading metadata information from LLVM IR. I couldn't find any good one stop source about how to write metadata to LLVM IR. Hence this post. Metadata API was introduced to LLVM IR in LLVM-2.7 (refer to [this](http://blog.llvm.org/2010/04/extensible-metadata-in-llvm-ir.html) post). 
Metadata information can be used to insert any kind of data related to code without any side-effects. It can be anything from debug information to any statistical data to some useless rant about how your day job sucks. For more information about them refer to the blog post in the link above.  Now let's see some code. All of these examples are written with LLVM-3.8. 

**Inserting Metadata**
Let's write a function pass that inserts total number of instructions as integer to metadata of the function and order number of instruction in the function to the instruction metadata as a string (just to demonstrate how to insert string as metadata). 

```
virtual bool runOnFunction(Function &F) {
      errs() << "I saw a function called " << F.getName() << "!\n";
      LLVMContext& C = F.getContext();
      int instructions = 0;
      if (!F.isDeclaration()) {
        for (auto I = inst_begin(F), E = inst_end(F); I != E; ++I) {
          instructions++;
          MDNode* N = MDNode::get(C, MDString::get(C, std::to_string(instructions)));
          (*I).setMetadata("stats.instNumber", N);
        }
        MDNode* temp_N = MDNode::get(C, ConstantAsMetadata::get(ConstantInt::get(C, llvm::APInt(64, instructions, false))));
        MDNode* N = MDNode::get(C, temp_N);
        F.setMetadata("stats.totalInsts", N);
      }
      return true;
    }
```

I think the code is pretty self explanatory. Important parts are setMetadata calls for [instruction](http://llvm.org/docs/doxygen/html/classllvm_1_1Instruction.html#a695a53ce0b9f537880373b4ea1824a6b) and [function](http://llvm.org/docs/doxygen/html/classllvm_1_1Function.html#ad97e10882bb3757cea47c798e6e6a2f4).  It takes a string which represents the kind of the metadata and the metadata node which has the information. Here is the whole [code](https://github.com/jiten-thakkar/llvm-pass-skeleton/blob/metadata/metadata/Metadata.cpp).

**Reading Metadata**

Reading metadata is very straight forward. Just use [getAllMetadata](http://llvm.org/docs/doxygen/html/classllvm_1_1Instruction.html#ad7aaaa756e911ad0624c4ad8660f6e69) to grab all metadata nodes or use [getMetadata(StringRef Kind)](http://llvm.org/docs/doxygen/html/classllvm_1_1Instruction.html#aafa29112cbe02e4adc9b36752c771991) to access a specific metadata node.  Here is full [code](https://github.com/jiten-thakkar/llvm-pass-skeleton/blob/metadata/metadata/ReadMetadata.cpp).

```
virtual bool runOnFunction(Function &F) {
      LLVMContext& C = F.getContext();
      SmallVector<std::pair<unsigned, MDNode *>, 4> MDs;
      F.getAllMetadata(MDs);
      for (auto &MD : MDs) {
        if (MDNode *N = MD.second) {
          Constant* val = dyn_cast<ConstantAsMetadata>(dyn_cast<MDNode>(N->getOperand(0))->getOperand(0))->getValue();
          errs() << "Total instructions in function " << F.getName() << " - " << cast<ConstantInt>(val)->getSExtValue() << "\n";
        }
      }
      for(auto I = inst_begin(F), E = inst_end(F); I != E; ++I) {
        //showing different way of accessing metadata
        if (MDNode* N = (*I).getMetadata("stats.instNumber")) {
          errs() << cast<MDString>(N->getOperand(0))->getString() << "\n";
        }
      } 
      return false;
    }
```
