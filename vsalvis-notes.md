# Vera's notes

## high level changes
### collect: everything is a flatMap
 - func: Index => DeliteCollection[A]
 - cond block is no longer separate but inside func
 - func is now properly guarded in generated code
 - map/zip and filter can be special-cased in codegen via extractors
 - ParFlat/ParBuffer architecture reworked
 - func and friends factored out into common supertype of all elems

### reduce: separated into reduce & fold
 - reduce:
 - no need to initialize accumulator (no zero block)
 - always strips first element regardless of type (can add overhead)
 - empty -> exception
 
 - fold:
 - unlike sequential fold, the initial element needs to be neutral or number of chunks will affect result
 - accumulator always initialized with neutral element (can be mutable or immutable), no need to guard
 - empty -> neutral
 - requires a second reduction function to be provided (to combine the partial results)
 - reduce with neutral element provided as a convenience class on top of fold
 - no longer a special ReduceTupleElem (can just use fold over a struct type)

### bucket ops: basically the same except valFunc & cond replaced with flatMap (todo)


### questions:
 - why is linear_buffer needed along with unknownSize? -- what are the exceptions to ParFlat == Map?
 - are we still using finalizer? 
 	- no
 - do we still need appendable separate from append? 
 	- no
 - flatMap: can allocation of intermediate collection be fused into write to output buffer? 
 	- need to do lowering
 - can/should fusion unapplies be generated for vector/matrix? (makes fusion less dependent on struct unwrapping) 
 	- try to do generically for structs


### todo:
 - remove singleton/empty from the public api for DeliteArray
 - do we want codegen nodes to be defined for singleton/empty?
 - rename hash->bucket and update to use multicollect 
 - bucketReduce vs bucketFold
