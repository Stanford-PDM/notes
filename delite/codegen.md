# Delite CodeGen

### Questions :
Generic codegen emits if(){}else{} => is it really generic ?

### phases for multiloops:

- alloc
- processRange
- init
- process (map)
- combine (tree-reduce)
- postCombine
- postProcInit
- postProcess (not sure)
- finalize (not sure)
- initAct

# Faq 

### what is the activation class or activation kernel ? 
a way to pass state around across different phases

### kernel numbers ? kernels with two numbers ?
Horizontally fused ->  produces 2 different results, once it returns, the runtime considers the results as 2 different entities

### resourceInfo ?
Locality info for runtime, (thread id ... ) not often used



# GPUGen

# MultiLoopSoa
func: Int => Struct