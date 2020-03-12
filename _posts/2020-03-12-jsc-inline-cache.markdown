---
layout: post
title:  "Inline cache implementation on JSC"
date:   2020-03-12 00:00:00 +0000
categories: [JSC]
---

The major goal of this article is to talk about how some parts of Inline Cache (IC) are implemented on JSC, explaining mainly its source code. We are focusing on "get" operations, but some of these concepts can be shared with types of IC that JSC implements, like "put" and "in" operations. If you are not familiar with IC, I recommend read the following material before continuing on this article:

- [JavaScript engine fundamentals: Shapes and Inline Caches](https://mathiasbynens.be/notes/shapes-ics): This gives a good background on how IC is used in V8.
- [ES6 Feature Complete](https://webkit.org/blog/6756/es6-feature-complete/): this explains some internals of how objects are represented on JSC and how it was possible to apply IC for properties located on prototype chain.
- [Inline Caching in JavaScriptCore](http://www.filpizlo.com/slides/pizlo-icooolps2018-inline-caches-slides.pdf): this is a presentation from Filip Pizlo about details of JSC IC, sharing numbers of benchmarks and giving detailed numbers of performance for a lot of variant of IC that JSC implements. 
- [inside javascriptcore's low-level interpreter](https://wingolog.org/archives/2012/06/27/inside-javascriptcores-low-level-interpreter): this is an article from Andy Wingo that explains what’s LLInt. I recommend reading that if you are not familiar with this kind of interpreter.
- [Optimizing Dynamically-Typed Object-Oriented Languages WithPolymorphic Inline Caches](http://bibliography.selflanguage.org/_static/pics.pdf): This paper introduced the idea of Polymorphic Inline Cache and has a very good section that explains IC on its background section. I recommend reading the entire paper if you want to understand Inline Caches better.

Now that we referenced material to basic concpets of Inline Cache, let's take a look on which parts of code the IC is implemented on JSC. We have separated this article in 2 sections. The first is going to discuss and show how IC is implemented on LLInt using `get_by_id` as example. The second part is going to discuss how IC is implemented on JIT compilers and which part of code is responsible to generate them.

## Inline cache on LowLevelInterpreter

Old versions of JSC used an interesting technique to implement Inline caches on LLInt. It changed bytecode while interpreting it, replacing an opcode by another after collecting feedback from execution. For example, on `get_by_id`, after its execution on slow path it was able to change opcode to `get_by_id_proto_load` or `op_get_array_length` (they will be described in more details below) whenever an access of this kind was made and such access was cachable. Also, information to perform fast path were stored on bytecode itself (there was space reserved there to store those kind of information).
However, since the introduction of [new bytecode format](https://webkit.org/blog/9329/a-new-bytecode-format-for-javascriptcore/), JSC bytecode is not writable anymore, which means that interpreter is not allowed to change itself while it is running. Given that, the new format uses a metadata table to store information collected during execution of each instruction. In that case, there is a `GetByIdModeMetadata` that stores information about an IC of a specific `get_by_id` opcode.

```
struct GetByIdModeMetadata {
    …
    union {
        GetByIdModeMetadataDefault defaultMode;
        GetByIdModeMetadataUnset unsetMode;
        GetByIdModeMetadataArrayLength arrayLengthMode;
        GetByIdModeMetadataProtoLoad protoLoadMode;
    };
    GetByIdMode mode;
    uint8_t hitCountForLLIntCaching;
};
```

As we can see above, there are multiple modes where JSC applies IC optimization. Let’s focus in the `GetByIdMode::Default` for now. The LLInt code to execute `get_by_id` is [here](https://github.com/WebKit/webkit/blob/df35fd9e088e4ff405c34704a6cbe9f61e756a6c/Source/JavaScriptCore/llint/LowLevelInterpreter64.asm#L1351). The part of code responsible to handle default mode is the following:

```
get_by_id:
...
    metadata(t2, t1)
    loadb OpGetById::Metadata::m_modeMetadata.mode[t2], t1
    get(m_base, t0)
    loadConstantOrVariableCell(size, t0, t3, .opGetByIdSlow)

.opGetByIdDefault:
    bneq t1, constexpr GetByIdMode::Default, .opGetByIdProtoLoad
    loadi JSCell::m_structureID[t3], t1
    loadi OpGetById::Metadata::m_modeMetadata.defaultMode.structureID[t2], t0
    bineq t0, t1, .opGetByIdSlow
    loadis OpGetById::Metadata::m_modeMetadata.defaultMode.cachedOffset[t2], t1
    loadPropertyAtVariableOffset(t1, t3, t0)
    valueProfile(OpGetById, t2, t0)
    return(t0)
...
.opGetByIdSlow:
    callSlowPath(_llint_slow_path_get_by_id)
    dispatch()
```

Now, let’s consider the following JS function:

```
function foo() {
    return o.f;
}

// Bytecode
foo#CwYqkR:[0x10b3c4130->0x10b3e1200, NoneFunctionCall, 14 (NeverInline)]: 6 instructions (0 16-bit instructions, 0 32-bit instructions, 1 instructions with metadata); 126 bytes (112 metadata bytes); 2 parameter(s); 8 callee register(s); 6 variable(s); scope at loc4
[   0] enter              
[   1] get_scope          loc4
[   3] mov                loc5, loc4
[   6] check_traps        
[   7] get_by_id          loc6, arg1, 0
[  12] ret                loc6
```

The very first time time `get_by_id` is executed, nothing is cached yet, and the check of structure ID on line 10 will fail, branching execution to `.opGetByIdSlow` and making a call to [llint_slow_path_get_by_id](https://github.com/WebKit/webkit/blob/df35fd9e088e4ff405c34704a6cbe9f61e756a6c/Source/JavaScriptCore/llint/LLIntSlowPaths.cpp#L754). Slow path operation is responsible to do the lookup of property into the object, but it also try to cache such lookup. The caching logic starts [here](https://github.com/WebKit/webkit/blob/df35fd9e088e4ff405c34704a6cbe9f61e756a6c/Source/JavaScriptCore/llint/LLIntSlowPaths.cpp#L767). This is the part where the mode of a `get_by_id` can change. The first part of the code, starting on line 772 and going until line 795 is a logic introduced to support [poly proto](https://trac.webkit.org/changeset/222827/webkit). The second “if” block (lines 799-812) is where we have the logic that caches “self” property access. This is the simplest IC case, since it requires cache of current StructureID and property offset to access it. Once metadata is filled, subsequent execution of `get_by_id` will be able to execute fast path for structures with the same StructureID of cached one. This makes access of monomorphic call sites very efficient.

### ProtoLoad access

While inline caching access of an own property in an object seems straight forward, that’s not the case for properties located on a prototype. After ES5, accessing properties stored on prototypes started to become very common on Array, Map and Set objects. Given that, being able to cache prototype access is likely to improve performance of modern JS programs. However, there are some issues when we cache access to properties of prototype. At first, we can cache the offset and base object where the property is present, however future execution of JS code can delete/modify such property, meaning that cached information is not valid anymore. Let’s see how this work on real code:

```
class C {
    method(){ return “C”;}
}
class B extends C {}
class A extends B {}

Function foo(o) {
    return o.method();
}

let a = new A;
for (…)
  foo(a); // returns “C”

B.prototype.method = () => “B”;
foo(a); // returns “B”, since B is before C on prototype chain.
```

When we consider the code above, we see a case where the access of `o.method` is changed because `B.prototype` was changed. Given that, IC of `o.method` needs to keep track if there is any change throughout the prototype chain. This is implemented on JSC using [Object Property Conditions and Adaptive Watchpoints](https://webkit.org/blog/6756/es6-feature-complete). Once we properly setup watchpoints on prototype chain, we are finally able to cache the ProtoLoad access. Different from “self” access, ProtoLoad mode needs to cache StructureID, offset and base object where property is present. The logic for this can be found on [setupGetByIdPrototypeCache](https://github.com/WebKit/webkit/blob/df35fd9e088e4ff405c34704a6cbe9f61e756a6c/Source/JavaScriptCore/llint/LLIntSlowPaths.cpp#L696). This also includes some logic for `UnsetMode`, where it is a special case when the property is not present throughout the prototype chain.

### ArrayLength mode

This mode is a special case for IC access to `arr.length` property. In this case, JSC stores the length of the array in a specific region, and it creates a IC case when this also happens.

## Inline Cache on JIT compilers

Previous section described some internals of LLInt IC implementation. Now we are going to explore the implementation of IC on JIT compilers. The major difference from LLInt is that JIT IC repatch generated code for observed cases and it supports [polymorphic inline cache](http://bibliography.selflanguage.org/_static/pics.pdf). On Baseline and DFG, IC not only plays a role on performance optimization, but it is also used to profile information about polymorphic call sites. The class [JITGetByIdGenerator](https://github.com/WebKit/webkit/blob/df35fd9e088e4ff405c34704a6cbe9f61e756a6c/Source/JavaScriptCore/jit/JITInlineCacheGenerator.h#L103) is important to make IC mechanism portable among Baseline, DFG and FTL layers. It contains a [StructureStubInfo](https://github.com/WebKit/webkit/blob/df35fd9e088e4ff405c34704a6cbe9f61e756a6c/Source/JavaScriptCore/bytecode/StructureStubInfo.h#L73) field, a class where all metadata and the business logic of IC policy is placed, including all counters used to decide if we should cache or give up an access, information of polymorphic cases that the IC is considering and pointers to each part of the code where IC is installed. When compiling Baseline code for `get_by_id`, the function [JIT::emit_op_get_by_id](https://github.com/WebKit/webkit/blob/df35fd9e088e4ff405c34704a6cbe9f61e756a6c/Source/JavaScriptCore/jit/JITPropertyAccess.cpp#L547) is called and it creates a `JITGetByIdGenerator`. It then calls `generateFastPath`, that generates a chunk of code with some noops (see [JITByIdGenerator::generateFastCommon](https://github.com/WebKit/webkit/blob/df35fd9e088e4ff405c34704a6cbe9f61e756a6c/Source/JavaScriptCore/jit/JITInlineCacheGenerator.cpp#L92)) and then jumps to slow path. Slow paths of JITed code are shared among all `get_by_id`, and this code is generated by [JIT::emitSlow_op_get_by_id](https://github.com/WebKit/webkit/blob/df35fd9e088e4ff405c34704a6cbe9f61e756a6c/Source/JavaScriptCore/jit/JITPropertyAccess.cpp#L600). See the generated code for snippet #1:

```
[   7] get_by_id          loc6, arg1, 0
          0x31dd93601940: mov 0x30(%rbp), %rax
          0x31dd93601944: test %rax, %r15
          0x31dd93601947: jnz 0x31dd936019f3
          0x31dd9360194d: jmp 0x31dd936019f3 // Jumps to slow path
          0x31dd93601952: o16 nop %cs:0x200(%rax,%rax)
          0x31dd93601961: o16 nop 0x8(%rax,%rax)
          0x31dd93601967: mov %rax, 0x10d0f33d8
          0x31dd93601971: mov %rax, -0x38(%rbp)
```

Essentially, the code for slow path calls into [operationGetByIdOptimize](https://github.com/WebKit/webkit/blob/df35fd9e088e4ff405c34704a6cbe9f61e756a6c/Source/JavaScriptCore/jit/JITOperations.cpp#L316) and interesting things start to happen.
`operationGetByIdOptimize` is responsible to perform a lookup of a property in a given object, but it also profiles and triggers IC generation on JITed code. Once `stubInfo->considerCaching` returns true, [repatchGetBy](https://github.com/WebKit/webkit/blob/df35fd9e088e4ff405c34704a6cbe9f61e756a6c/Source/JavaScriptCore/jit/Repatch.cpp#L414) is executed and it calls [tryCacheGetBy](https://github.com/WebKit/webkit/blob/df35fd9e088e4ff405c34704a6cbe9f61e756a6c/Source/JavaScriptCore/jit/Repatch.cpp#L182). This part of code considers a lot of cases to be inlined, including `Array.length`, `String.length`, `Direct/ScopedArguments` access and etc. Each case has its special operations to perform fast access, but we will take a deep look into [self access](https://github.com/WebKit/webkit/blob/df35fd9e088e4ff405c34704a6cbe9f61e756a6c/Source/JavaScriptCore/jit/Repatch.cpp#L263) for now. As we do on LLInt, self property access IC needs to perform a type check (i.e compare StructureID) and if it matches, we access property using cached offset. The code generated for self access is on [InlineAccess::generateSelfPropertyAccess](https://github.com/WebKit/webkit/blob/df35fd9e088e4ff405c34704a6cbe9f61e756a6c/Source/JavaScriptCore/bytecode/InlineAccess.cpp#L179). See below an example of code generated for a monomorphic self access: 

```
Generated JIT code for InlineAccessType: 'property access':
    Code at [0x31dd9360194d, 0x31dd9360194d):
      0x31dd9360194d: cmp $0xfa72, (%rax) / /StructureID check
      0x31dd93601953: jnz 0x31dd936019f3 // jumps to slow path if check fails
      0x31dd93601959: mov 0x10(%rax), %rax // Access directly from offset
      0x31dd9360195d: o16 nop %cs:0x200(%rax,%rax)
```

It is important to see that the code above is patched into the existing code for `get_by_id` (check the address `0x31dd9360194d` is where execution was branched to slow path before).

Another aspect of IC on JIT layers is the fact that they are more powerful than LLInt, given they are also applied to getter, setters and for `PureForwardingProxyType` (this is the case of “global” object). This mechanism is not only used to generate faster code, but FTL and even DFG rely on the information profiled by IC to inline structure checks and offset access into their IR, allowing more aggressive optimizations to be applied (check more detailed information about this on this [article](https://webkit.org/blog/3362/introducing-the-webkit-ftl-jit/)), like inlining a getter or a setter.

The polymorphic inline cache version is compiled considering information stored on all cases added to it. The code that starts IC generation is on [PolymorphicAccess::regenerate](https://github.com/WebKit/webkit/blob/df35fd9e088e4ff405c34704a6cbe9f61e756a6c/Source/JavaScriptCore/bytecode/PolymorphicAccess.cpp#L390) that will then call [AccessCase::generateImpl](https://github.com/WebKit/webkit/blob/df35fd9e088e4ff405c34704a6cbe9f61e756a6c/Source/JavaScriptCore/bytecode/AccessCase.cpp#L1362) for each case. An example of polymorphic code with 2 cases of structure can be seen bellow:

```
Generated JIT code for Access stub for foo#CwYqkR:[0x10fdc4130->0x10fde1200, BaselineFunctionCall, 14 (NeverInline)] bc#7 with return point CodePtr(0x41d4c50018b5): 
Load:(Generated, ident = 'f', offset = 1, structure = 0x10fdcd020:[0x2eec, Object, {g:0, f:1}, NonArray, Proto:0x10fdf4000, Leaf], viaProxy = false, additionalSet = 0x0), 
Load:(Generated, ident = 'f', offset = 0, structure = 0x10fdcd080:[0x37f4, Object, {f:0}, NonArray, Proto:0x10fdf4000, Leaf], viaProxy = false, additionalSet = 0x0):
    Code at [0x41d4c5001b00, 0x41d4c5001b40):
      0x41d4c5001b00: mov (%rax), %esi
      0x41d4c5001b02: cmp $0x37f4, %esi // This check for the first Structure
      0x41d4c5001b08: jnz 0x41d4c5001b17 // If fails, falls back to second check
      0x41d4c5001b0e: mov 0x10(%rax), %rax
      0x41d4c5001b12: jmp 0x41d4c50018b5
      0x41d4c5001b17: cmp $0x2eec, %esi // This is where second structure check happens
      0x41d4c5001b1d: jnz 0x41d4c5001929 // if it fails, branch to slow path
      0x41d4c5001b23: mov 0x18(%rax), %rax
      0x41d4c5001b27: jmp 0x41d4c50018b5 
      ...
```

The code on `AccessCase::generateImpl` is very dense, since it considers a lot of cases where we can perform a fast path, but most of them follows the template of doing some checks and properly accessing information requested. 

## Conclusion

The goal of this article is to share some of my knowledge about internals of JSC. The inline cache mechanism is one of the part of the code base that took me a lot of effort to get used to, and I’m still learning some parts of it, since it keeps changing. I think that pointing most of papers and articles I used to understand this mechanism can be used as shortcut for new developers into the code base. If you have any question, comment or improvements to share, please feel free to contact me over my email or [Twitter](https://twitter.com/caio_nepo).
