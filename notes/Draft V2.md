

# Scalable and Modular Verified Compilation via Automated Proofs for Declarative Transformations

A fully end-to-end verified *mainstream* compiler with a small trusted code base remains somewhat of an unattainable holy grail. End-to-end verified compilers do exist (e.g., CompCert [[@leroyCompCertFormallyVerified]] and CakeML [[@kumarCakeMLVerifiedImplementation2014]]), but the extra verification burden makes compiler development challenging, relegating such compilers to specific high-assurance niches. On the other hand, there are tools which can fully automatically prove the correctness of a specific compiler transformation (e.g., Alive [[@lopesAlive2BoundedTranslation2021]], which has gained widespread adoption in the LLVM community, and [[@armstrongIslaIntegratingFullScale2021]]), but such tools generally rely on SMT solvers to achieve their push-button automation, making them unsuited in our quest for a minimal trusted code base. I wish to investigate a middle road: can we verify the core transformation algorithm separately from specific transformation instances, and then develop proof automation for fully automatically showing local correctness of a transformation in a way that composes into a proof of global correctness?

Specifically, I'll develop two core components of a verified compiler: an optimisation pass based on peephole optimisations and a code generator that lowers the compiler IR (Intermediate Representation) produced by the optimiser into assembly. Of course, a fully end-to-end compiler would need many more components, such as a verified front-end that parses source code and transforms it into semantically equivalent compiler IR, however such components will be out-of-scope for my PhD.

To ensure a minimal trusted code base we'll develop our proofs and proof automation in Lean. Besides a modern theorem prover, Lean is also aiming to be a general purpose functional programming language. It is certainly not the first theorem prover that allows its users to extract executable code from a formalization, but Lean goes several steps further: the Lean compiler is itself in large parts written in Lean. Furthermore, proof automation for Lean is also written in Lean itself. Combined, this makes it a compelling platform for verified compilation. On the one hand, Lean's rich logic enable us to finely model the semantics of programs. On the other, Lean itself aims to be a language that is suited for large-scale software, which necessitates a modern optimising compiler backend, but, Lean also has a feature that enables the use of *compiled* decision procedures in proofs. Using this feature makes the optimiser part of the trusted codebase for our proofs, and it is thus crucial for the logical soundness of our system that mis-compilations do not occur.

>[!TODO]
> Add a sentence to the above to conclude, e.g., by saying that we imagine a future where Lean has a fully verified compiler backend written in Lean.
> Also, do we want to move this paragraph lower down, as it might be more relevant to expand on the mayor goals first, before talking about our choice of ITP.


For the optimisation pass, we choose to operate on LLVM IR --- the same compiler IR used by Clang, Swift and Rust. Although others have formalised the semantics of LLVM IR in a proof assistant before ([[@zhaoFormalVerificationSSABased]], [[@zakowskiModularCompositionalExecutable2021]]), those efforts included no proof automation. This will to our knowledge be the first effort to provide fully automated proofs for correctness of LLVM optimisations in a way that doesn't have to trust an external solver (as Alive does [[@lopesAlive2BoundedTranslation2021]]). Removing this dependency is important as SMT solvers are large, complex pieces of software with a history of bugs ([[@brummayerFuzzingDeltadebuggingSMT2009]], [[@mansurDetectingCriticalBugs2020]]). 

The optimisation pass will be instantiated with a set of *peephole optimisations*, consisting of program patterns to match for and corresponding program patterns to rewrite to. Lean-MLIR ([[@bhatVerifyingPeepholeRewriting2024]]), a project in development at my research group which I've contributed to, currently has a prototype of such an optimiser. However, the current version is only able to apply *pure* optimisations. I wish to expand this to an optimiser that can apply optimisations involving side-effect such as memory loads/stores, immediate undefined behaviour, or even potentially non-terminating control flow, and, crucially, provide push-button proof automation for such side-effecting optimisations, using symbolic execution.

>[!TODO]
> Maybe mention InstCombine in LLVM?
>If we do switch to MLIR dialects, here I could add the argument that MLIR dialect promote high-level structure, which makes more transforms possible as local peepholes.


After optimisation, the next phase will be to generate assembly from our optimised IR and, crucially, to verify that the generated assembly has the same semantics as the optimised IR. For maximum trust, we'd like to use authoritative models of the ISA semantics of whichever platform we are compiling for to give semantics to the assembly program. Sail ([[@armstrongISASemanticsARMv8a2019]]) provides such models: the Sail model for Arm-A is automatically derived from the official ASL reference, and their RISC-V model has been adopted by the RISC-V Foundation. We'll phrase the code generator in a similar way: by having fragments of compiler IR to match for, and corresponding fragments of assembly to rewrite them with. Still, code generation presents some unique challenges. Firstly, although the transformations themselves should be simpler, the Sail semantic model will be much larger than our handwritten LLVM semantics. Secondly, we have to deal with semantics of different languages (the compiler IR and the assembly language). It is likely that proof automation for code generation will thus need a slightly different design than proof automation for peephole optimisations.

>[!TODO]
>How do actual compilers do code gen? Is it just peephole-like transformations, or do they do something more sophisticated


Finally, to model the semantics of LLVM IR in a modular way, we'd like to follow Zakowski et al. ([[@zakowskiModularCompositionalExecutable2021]]) and use interaction trees ([[@xiaInteractionTreesRepresenting2020]]). Unfortunately, interaction trees are *coinductive* in nature, to capture possibly non-terminating programs, and Lean currently lacks support for coinductive type. Therefore, I'd like to continue the work that I started in own master's thesis [[@keizerImplementingDefinitionalCo]], which is based on earlier work by Avigad et al. ([[@avigadDataTypesQuotients2019]]), and develop a Lean library for coinductive types.


My core contribution is as follows:
* A proof-producing translation validation tool for compiler optimizations, which works on a higher-level IR with relatively straightforward formalized semantics, but is able to analyse complex transformations.
* A proof-producing translation validation tool for code generation using large-scale authoritative ISA models. Here the transformations are relatively simple, but the complexity of the instruction semantics presents unique challenges.
* Finally, we develop technical infrastructure needed for both of these tools in the form of coinductive types support in Lean.


Note that the benefits of a verified compilation pipeline go far beyond simply reducing the chance of bugs. We believe it will lead to a culture shift in the way software is written. Traditionally, writing compiler optimizations was a domain reserved for the wise elders who we could trust to perform the appropriate "black magic". Reasoning about the correctness of a transformation was simply too hard, so we had to rely on interpersonal trust. Tools like Alive have already caused a shift in this approach: nowadays PRs that propose adding a new peephole optimisation to LLVM can include a link to Alive's online interface showing the correctness, meaning no further discussion or review is required. However, this effect is still limited to making reviews for proposed additions to LLVM easier.

Lean, again, presents a very interesting opportunity because of it's *extensibilty*: Lean already has an extensible syntax and a powerful meta-programming API. In fact, a lot of features that are considered part of Lean's core are actually implemented using the same meta-programming facilities that are available to libraries (we'll use this extensively in ch. 3). However, we imagine a future where this extensibility could extend to the backend of the compiler also, such that a library writer might implement an algorithm in a functional style that is easy to reason about, but also provide an optimized assembly *and* a proof that the assembly is equivalent. Traditionally, hard-coded assembly in a library has been cause for increased scrutiny. Optimizing is hard, do we really trust the author of this library to have done it correctly? Verification means we no longer have to trust, instead we get to have fearless optimization!








>[!TODO]
>How much more do I need?
>Of course, there is the month-by-month workplan, but what else?
>Presumably, a more thorough overview of related work?
>Do I need more technical detail on a chapter by chapter basis.







## Translation Validation for MLIR (a.k.a. the optimisation pass)

LLVM IR is an intermediate representation that sits just above the level of abstraction of assembly, and has proven a successful platform to perform optimisations on before finally lowering to platform-specific assembly. LLVM has been used as a back-end for many mainstream compilers, such as Clang, Rust and Swift.

A crucial property of LLVM IR, is that it is in static single assignment (SSA) form: rather than operating on a fixed set of registers, LLVM IR has variables that are assigned exactly once. This form makes it easier to apply peephole rewrites, as we can simply match for a pattern along what is called the def-use chain, even if there are unrelated instructions in between.

Now, although LLVM is a bit higher level than assembly, it's still low-level and, e.g., features unstructured control flow. The LLVM community realised that having more structure available would enable even more optimisations to be done purely locally (i.e., as peephole rewrites). Thus, MLIR (Multi Level Intermediate Representation) was born: a framework to define compiler IRs at various levels of abstraction. MLIR mandates that IRs are in SSA form, and takes care of syntactic overhead. We can instantiate the dialect with a specific set of operations, called a dialect. Generally, a copmiler IR is made up of a combination of such dialects. For this chapter, we shall focus on an IR that contains bitvector arithmetic (`arith` dialect) memory manipulations (`memref` and `ptr` dialects) and structured control flow (i.e., for and while loops, `scf` dialect). This is still fairly low level, but the use of structured control flow puts us just above the level of abstraction of LLVM IR.

>[!TODO]
>- [ ] Tobias mentioned the [`ptr`](https://mlir.llvm.org/docs/Dialects/PtrOps/) dialect also, but the documentation for that dialect is rather sparse, and in particular doesn't mention it's operations? Tobias mentioned the difference is that memref has tensors and such, whereas pointer deals with raw pointers. I can't really see this in the memref docs, though, I wonder if it's something like memref contains the basic operations, but not the types, so there is `memref+tensor` or `memref+ptr`, i.e., `ptr` doesn't have operations because it's not meant to be used by itself?
>- [ ] I wonder if there's a lowering from `memref`/`ptr` to LLVM IR that can clear this up


### Lean-MLIR Today

Lean-mlir is a framework for modelling the *semantics* of MLIR dialect in Lean, developed by my research group [[@bhatVerifyingPeepholeRewriting2024]], focusing on the *pure* (i.e., side-effect-free) fragment of MLIR dialect. 

In particular, lean-mlir currently has a model of just the bitvector arithmetic operations of LLVM IR, and a verified peephole rewriter that can apply a pure peephole rewrite (i.e., neither the program fragment being matched on nor the fragment being rewritten to contain side-effecting operations) and, assuming the local correctness of every rewrite, we've shown the rewriter preserves the global semantics.

Furthermore, the Lean FRO has, in collaboration with my research group and myself, been developing proof automation for deciding the equality of fixed-width bitvector expressions \[CITATION NEEDED]. In particular, this proof automation allows us to fully automatically decide the local correctness of *pure* optimisations. Note that although this tactic employs an external (certificate-producing) SAT solver, it also includes a verified certificate checker, so that the final proofs do *not* rely on the correctness of the solver. That said, the checker would be too slow if ran inside the Lean kernel, so to get decent performance, this tactic does rely on the previously mentioned feature to run the compiled version of a Lean, thus expanding the trusted code base to include the Lean compiler.

### Proof Automation for Side Effects

I wish to expand upon this first by modelling all operations of the dialect mentioned before. The bitvector arithmetic fragment of the previous section is basically the same as `arith`, and the framework also already models part of `scf` (in particular, it models counting for-loops, but not while-loops with arbitrary conditions yet), so the main missing piece is a model of memory and memory operations. To make this easier, I'll focus purely on single-threaded code, foregoing the need for complicated memory models.

The main innovation in this project will be to build proof automation in Lean for translation validation on this compiler IR. That is, given a source and target program (potentially including holes), will automatically decide the correctness of the transformation. To do this in the presence of side-effects, I'll build a symbolic simulator.


>[!TODO]
>Do I mention my work on LNSym here?



>[!TODO]
>- [ ]  Read/skim through Alive. I should briefly contrast what I propose to do here with Alive's approach (which is not applicable because it uses SMT solvers).
>- [ ]  How much detail do I need to give about how I propose to build a symbolic simulator (e.g., whether it's proof-producing metacode vs a verified simulator a la katamaran)



### Refinement and Undefined Behaviour

So far, we've talked about semantic equivalence as the condition for correctness of an optimisation, but in reality it's a bit more complicated because of the notion of undefined behaviour (UB). In short, if a program has UB, it's legal to change the program to anything. Conversely, if a source program is fully defined, then it's illegal to introduce UB in the target. The corresponding relation between programs is called *refinement*.

>[!TODO]
> Does `arith` have the same UB/poison semantics as LLVM IR? Does it matter, or can we just formalize it as if it did?

>[!TODO]
>I wanted to talk about changing the verified peephole rewriter to work with refinement in this section, rather than straight equivalence, but maybe that's not relevant since we've not talked about changing it to work with side-effects either.


### Related Work

We can broadly categorise earlier efforts to apply formal methods to LLVM into those that formalise the semantics of LLVM, but perform manual proofs (e.g., VeLLVM [[@zhaoFormalVerificationSSABased]], [[@zhaoFormalizingLLVMIntermediate2012]] or work by Zakowski et al. [[@zakowskiModularCompositionalExecutable2021]]) and tools like Alive ([[@lopesAlive2BoundedTranslation2021]]) that perform fully automated translation validation, but rely on SMT solvers. 

More broadly, CompCert ([[@leroyCompCertFormallyVerified]]) and CakeML ([[@kumarCakeMLVerifiedImplementation2014]]) are examples of fully end-to-end verified compilers. More recent work by Mullen et al. [[@mullenVerifiedPeepholeOptimizations2016]] argues for peephole rewriting as a way to lower proof burden in CompCert development, but they still prove these rewrites manually.

Katamaran [[@keuchelVerifiedSymbolicExecution2022]] is a symbolic simulation tool written in a different ITP. However, this project focusses on being able to run the simulator outside of the ITP environment, hence eschewing meta-programming. Our main purpose is modular proof automation, so being tied to Lean is no problem for us, allowing us to make different implementation choices.








>[!WARNING]
>Old notes ahead








## Translation Validation for LLVM
* (Properly introduce LLVM IR, motivate why it's cool)
* LLVM IR has been formalized in a theorem prover before, in Vellvm, but that project crucially only gave a semantics and verified concrete transformation passes; they did no proof automation at all!

>[!NOTE]
>Matthieu made an interesting point, that a combination of higher-level MLIR dialect like memref+scf+arith might make for a more compelling target, since it would eliminate the need to reason about jumps/branches. I'll have to implement this for Arm later on anyway, but maybe picking a structured target in this chapter here might make for a more interesting/diverse story.


### SSA
* (Briefly introduce SSA-form)
* (Introduce MLIR)
* (Explain basic setup of LeanMLIRs datastructures)
* (Introduce **poison**, as a value-level/SSA-obeying construct for *deferred* UB)

### Side-effects: memory read/writes and branches
>[!TODO] Task
>Define the right monads we can use to talk about LLVM IR's side-effects:
>* Memory
>* Immediate UB
>* Control flow: jumps & branches

* LeanMLIR models side-effect as monadic effects)
* We will only model a sequential memory model: reads/writes modify a `StateM Memory` monadic state 
* Similarly, we can model unstructured control flow as side-effect (see [[Terminators and the Goto Monad]]).
* Also: immediate UB is a side-effect!

### Proof Automation: The Pure Case
* (Talk about the tactics we have to eliminate framework syntactics overhead into pure expressions)
* (Talk about `bv_decide` & friends to decide said pure expressions)

### Symbolic Simulation
>[!TODO] Task
>Implement [[Symbolic simulation for LLVM IR]]:
>- [ ] First, expand our model of LLVM IR to include immediate UB as a monadic effect
>	- [ ] Write a tactic that can derive an expression which describes exactly when a program has UB
>		* This will allow us to hoist out the UB-checks from all individual operations to the top-level and rewrite the program semantics to  `if /- program has UB -/ then throw_UB_effect else /- pure bitvector semantics, with UB checks simplified out -/`
* To automate reasoning about side-effecting programs, we employ symbolic simulation
* The goal is to be competitive with Alive in performance, but we'll need to actually justify all of the performance optimizations we do.
* Alive uses some very clever, but non-obvious tricks to squeeze more performance out of the SMT solver. Our translation validation will thus not only eliminate the need to trust in the SMT solvers implementation, it will also increase the confidence that the tricks Alive uses are justified.
* Furthermore, it allows us to go beyond what pure automation can do by gracefully introducing manual input. For example, Alive only handles loops that can be proven to terminate after a limited number of unrollings. Meanwhile, we could enhance our symbolic simulator to translate an infinite loop into a *coinductive* trace (see ch. 3 for more details).
* (Talk about LNSym as a codebase that this effort will be based on)


## Symbolic Simulation for Arm
> Mention LNSym, talk about Isla/Islaris

The [[Islaris]] project aims to enable foundational reasoning about low-level programs in Coq by utilizing authoritative ISA models from the [[Sail]] ecosystem. A critical component of the workflow is Isla, an SMT solver-based symbolic simulator that takes in a program and an ISA semantics, and produces a trace that models the behaviour of the original program. 

Given that SMT solvers are large, complex programs prone to bugs [[@brummayerFuzzingDeltadebuggingSMT2009]] [[@mansurDetectingCriticalBugs2020]], Islaris includes proof automation to perform translation validation, which attempts to prove that the trace produced by Isla is correct. Unfortunately, this translation validation was found infeasible for Armv8-A, as "the size of the model means it cannot be manipulated efficiently in Coq" [[@sammlerIslarisVerificationMachine2022]].

To enable high-confidence end-to-end proofs even for Armv8-A, I thus want to investigate writing a proof-producing symbolic simulator for Sail.

### Rough Plan

The desired goal is to be able to symbolically simulate Arm programs, using the authoritative semantics, without having to trust an SMT solver. 

However, this is a rather large goal, so we'll focus on a simpler fragment of programs (e.g., user-mode only, or programs that use only a limited set of instructions). Then, we evaluate by comparing performance of IsLean with Isla on a set of benchmark programs in the fragment. From this point, I can iterate by expanding the fragment of supported stuff and optimizing, each time comparing performance against Isla. 


### Risks

This project heavily depends on the translation from Sail to Lean, which is still very much in development. There's enough interest from enough parties that it is highly likely this effort will succeed, but the exact timeline of when it is possible to translate the full Armv8-A model into Lean is not clear to me at the moment.

### Non Goals

We'll limit ourselves to single-threaded executions, which should make dealing with memory accesses considerably simpler.



## Coinductive Types

* To properly model infinite traces of events, we need coinductive types
* Lean currently lacks coinductive types
* [[QpfTypes]] to the rescue!






> [!TODO]
> - [ ] Make the transition from ch 1 to ch 2 smoother, or at least, make the connection between these chapters clearer,
> - [ ] Explicate the "grand vision" that connects together ch. 1 and ch. 2 (ch. 3 is fine, there is a justification for why we need coinduction) in the introduction,
> - [ ] Flesh out ch 3 more



Working Title: "Scalable and Modular Verified Compilation Via Proof Producing Symbolic Simulation"