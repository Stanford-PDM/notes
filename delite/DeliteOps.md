# Unanswered Questions:
### Is there a better way to mirror than through `original` ?


#LMS

## AbstractLoop
```scala
abstract class AbstractLoop extends Def[A] with CanBeFused {
    type A

    val size: Exp[Int] // Size of the loop
    val v: Sym[Int] // Symbol for index inside the body
    val body: Def[A] // Expression representing the body of the loop
}
```
**TODO: why does `AbstractLoop[A]` extends `Def[A]` while body is also `Def[A]` ?**

#Delite

## Introduction

**TODO: Verify assumptions**

The DeliteOpLoop is the base of the hierarchy of loops that are available to the DSL creator. 

The DeliteLoopElem represents the body of a loop, and is useful only to extract the body of loops from the deliteOps.

`DeliteOpsIR` -> internal to the delite compiler. (`DeliteLoopElem`lives here)

`DeliteOps` -> operations available to DSL authors. (`DeliteOpLoop`'s childrens lives here)

## Legend

![Alt text](http://www.dotty.ch/g?
  digraph G {
    "LMS Code" [color=gray,style=filled];
    "DeliteOps.scala" [color=lightblue,style=filled];
    "DeliteOpsIR.scala" [color=salmon,style=filled];
  }
)

## DeliteLoops

![Alt text](http://www.dotty.ch/g?
  digraph G {
    rankdir=BT;
    Def [shape=box,color=gray,style=filled];
    ;
    AbstractLoop [shape=box,color=gray,style=filled];
    AbstractLoop -> Def;
    ;
    DeliteOp [shape=box,color=salmon,style=filled];
    DeliteOp -> Def;
    ;
    DeliteOpLoop [shape=box,color=salmon,style=filled];
    DeliteOpLoop -> AbstractLoop;
    DeliteOpLoop -> DeliteOp;
    ;
    DeliteOpCollectLoop [shape=box,color=lightblue,style=filled];
    DeliteOpCollectLoop -> DeliteOpLoop;
    ;
    DeliteOpFlatMapLike [shape=box,color=lightblue,style=filled];
    DeliteOpFlatMapLike -> DeliteOpCollectLoop;
    ;
    DeliteOpFoldLike [shape=box,color=lightblue,style=filled];
    DeliteOpFoldLike-> DeliteOpCollectLoop;
    ;
    DeliteOpReduceLike [shape=box,color=lightblue,style=filled];
    DeliteOpReduceLike -> DeliteOpCollectLoop;
    ;
    DeliteOpForeach [shape=box,color=lightblue,style=filled]; 
    DeliteOpForeach -> DeliteOpLoop
    ;
    DeliteOpHashCollectLike [shape=box,color=lightblue,style=filled];
    DeliteOpHashCollectLike -> DeliteOpLoop;
    ;  
    DeliteOpHashReduceLike [shape=box,color=lightblue,style=filled];
    DeliteOpHashReduceLike -> DeliteOpLoop;
    ;
    DeliteOpMapLike [shape=box,color=lightblue,style=filled];
    DeliteOpMapLike -> DeliteOpFlatMapLike;
    ;
    DeliteOpMapI [shape=box,color=lightblue,style=filled];
    DeliteOpMapI -> DeliteOpMapLike;
    ;
    DeliteOpFilterI [shape=box,color=lightblue,style=filled];
    DeliteOpFilterI -> DeliteOpFlatMapLike;
    ;
    DeliteOpFlatMapI [shape=box,color=lightblue,style=filled];
    DeliteOpFlatMapI -> DeliteOpFlatMapLike;
  }
)

## DeliteElems
![Alt text](http://www.dotty.ch/g?
  digraph G {
    rankdir=BT;
    Def [shape=box,color=gray,style=filled];
    ;
    DeliteLoopElem [shape=box,color= salmon,style=filled];
    ;
    DeliteHashElem [shape=box,color=salmon,style=filled];
    DeliteHashElem -> Def;
    ;
    DeliteHashIndexElem [shape=box,color=salmon,style=filled];
    DeliteHashIndexElem -> DeliteHashElem;
    DeliteHashIndexElem -> DeliteLoopElem;
    ;
    DeliteCollectBaseElem [shape=box,color=salmon,style=filled];
    DeliteCollectBaseElem -> Def;
    DeliteCollectBaseElem -> DeliteLoopElem;
    ;
    DeliteFoldElem [shape=box,color=salmon,style=filled];
    DeliteFoldElem -> DeliteCollectBaseElem;
    ;
    DeliteReduceElem [shape=box,color=salmon,style=filled];
    DeliteReduceElem -> DeliteCollectBaseElem;
    ;
    DeliteCollectElem [shape=box,color=salmon,style=filled];
    DeliteCollectElem -> DeliteCollectBaseElem;
    ;
    DeliteHashReduceElem [shape=box,color=salmon,style=filled];
    DeliteHashReduceElem -> DeliteHashElem;
    DeliteHashReduceElem -> DeliteLoopElem;
    ;
    DeliteHashCollectElem [shape=box,color=salmon,style=filled];
    DeliteHashCollectElem -> DeliteHashElem;
    DeliteHashCollectElem -> DeliteLoopElem;
    ;
    DeliteForeachElem [shape=box,color=salmon,style=filled];
    DeliteForeachElem -> Def;
    DeliteForeachElem -> DeliteLoopElem;
  }
)

## DeliteOp
```scala
trait DeliteOp extends Def[A]{
    type A
    
    def original: Option[(Transformer,Def[_])] = None
}
```
The base class for all of the delite operations. 

The original field is used to keep a pointer to the version of a node prior to a transformer pass.   It is `None` the first time the node is created, otherwise it's a pointer to the previous version of the node plus the transformer that was run.

**(TODO: format) (TODO: understand how this interacts with mirroring and CSE)**

It is used to call transformers on internal stuff in the node 
so a delite op can contain a body, for example
which isn't included in it's mirroring rule
so during its constructor, the delite op actually checks if original is defined, then copies+mirrors the previous version of the body from the original using the pointer to the transformer

Everytime there is something like this :

```scala    
lazy val field: Type = copyTransformedOrElse(_.field)(someValue).asInstanceOf[Type]
```

It means that the field's value is `someValue` which is either a default value or another field in the class. And the reason why it is written this way is to automatically apply a transformer to all members of a class. 

For consiceness reasons, from this point on, I will rewrite the above as :

```scala
val field: Type = someValue
```

## DeliteOpLoop
```scala
class DeliteOpLoop extends AbstractLoop[A] with DeliteOp[A] {
    type A:Manifest

    val v: Sym[Int] = fresh[Int] // Symbol for index inside the body
    val numDynamicChunks: Int = 0 // Parallellism of the loop
}
```
Base of all the delite parallel loops. 

## DeliteOpCollectLoop
```scala
class DeliteOpCollectLoop[O:Manifest, R:Manifest] extends DeliteOpLoop[R] {
    // The flatmap function, creating a collection for every loop index
    // that is then further processed (the loop index is this.v).
    def flatMapLikeFunc(): Exp[DeliteCollection[O]]

    // FlatMap loop bound vars
    val iFunc: Exp[DeliteCollection[O]] = flatMapLikeFunc()
    val iF: Sym[Int] = fresh[Int]
    val eF: Sym[DeliteCollection[O]] = fresh[DeliteCollection[O]](iFunc.tp)
}
```

Most parallel ops have loop bodies ("elems") that create an intermediate collection for every loop index. For a flatMap loop, those collections are concatenated, for a reduce loop, they are reduced. This general structure allows for a single operation to encode combinations of map, filter and flatMap operations. For example, a flatMap and a mapReduce can be combined by fusion into a single reduce node which contains a flatMap.  

The elems extend DeliteCollectBaseElem and the ops extend this trait to get the necessary attributes. The intermediate collection elements have type O, the result of the loop has type R.


**TODO: what is iF and eF ????**


## DeliteLoopElem
```scala
trait DeliteLoopElem {
    val numDynamicChunks: Int // Parallelism of the loop
}
```

Base class for all of the loop bodies of delite.

**TODO: why doesn't `DeliteLoopElem` extend `Def` ?**

## DeliteCollectBaseElem
```scala
class DeliteCollectBaseElem extends Def[O] with DeliteLoopElem {
    type A:Manifest // Type of element of intermediate collection
    type O:Manifest // Type of result
    
    // flatmap function, produces an intermediate collection for each
    // iteration,  which is appended one by one to the output collection
    // through an inner loop
    val iFunc: Block[DeliteCollection[A]]
    
    // true in the general flatMap case, false for a fixed-size map
    val unknownOutputSize: Boolean
    
    // The number of dynamic chunks
    val numDynamicChunks: Int
    
    // bound symbol to hold the intermediate collection computed by iFunc
    val eF: Sym[DeliteCollection[A]]
    
    // inner loop index symbol to iterate over the intermediate collection
    val iF: Sym[Int]
    
    // size of the intermediate collection (range of inner loop)
    val sF: Block[Int]
    
    // element of the intermediate collection at the current inner loop index
    val aF: Block[A]
}
```

**TODO: what are thos names: ???**
**iFunc -> intermediate function ?**
**eF -> expr Flat ?**
**sF -> expr FlatMap ?**
**aF -> expr FlatMap ?**

This LoopElem is the base for all operations that produce an intermediate collection. 

For example, a flatMap operation will directly output the concatenation of the intermediate collections, whereas a reduce operation will reduce them first. The reduce operation also has an intermediate collection (it is thus really a flatMap-reduce) so it can be fused with previous operations. The flatMap function might actually define a simple map or a filter, which allow more efficient code to be generated because the intermediate collection doesn't need to be allocated and iterated over. Those special cases are represented by their respective DeliteCollectType (see below) and are handled at the codegen level.


## DeliteCollectElem
```scala
class DeliteCollectElem extends DeliteCollectBaseElem[A, CA] {
    type A:Manifest // Type of element of the intermediate collection
    type I <: DeliteCollection[A]:Manifest // Type of intermediate collection
    type CA <: DeliteCollection[A]:Manifest // Type of output collection
    
    // The output collection/buffer to be used
    val buf: DeliteCollectOutput[A,I,CA]
    
    override val iFunc: Block[DeliteCollection[A]]
    override val unknownOutputSize: Boolean
    override val numDynamicChunks: Int
    override val eF: Sym[DeliteCollection[A]]
    override val iF: Sym[Int]
    override val sF: Block[Int]
    override val aF: Block[A]
}
```
  
This LoopElem is used for all flatMap-type operations (loops that produce an output collection of type CA). There are two different output strategies, see DeliteCollectOutputStrategy. 

## DeliteOpFlatMapLike
```scala
class DeliteOpFlatMapLike extends DeliteOpCollectLoop[O,CO] {
    type O:Manifest // type of individual element of resulting collection
    type I <: DeliteCollection[O]:Manifest // type of intermediate colleciton
    type CO <: DeliteCollection[O]:Manifest // type of resulting collection

    // Behavior supplied by subclasses
    
    // Allocates the intermediate result collection. The size passed is 0 if
    // the OutputStrategy is OutputBuffer, and it is op.size (the input size of
    // this loop) if it is OutputFlat.
    def alloc(size: Exp[Int]): Exp[I] = ??? // should be overriden
    // the finalizer to transform the intermediate result collection of type I
    // to the final output collection of type CO
    def finalizer(x: Exp[I]): Exp[CO]
    // true in the general flatMap case, false for a fixed-size map where the
    // output size is known before runtime
    val unknownOutputSize = true

    // Buffer bound vars
    val eV: Sym[O] = fresh[O] // TODO: element of the input collention ?
    val sV: Sym[Int] = fresh[Int] // TODO: size of the input collection ?
    val allocVal: Sym[I] = reflectMutableSym(fresh[I]) // TODO: ???
    val iV: Sym[Int] = fresh[Int] // TODO:  ???
    val iV2: Sym[Int] = fresh[Int] // TODO: ???
    val aV2: Sym[I] = fresh[I] // TODO: ???

    // buffer elem
    lazy val buf: DeliteCollectOutput[O, I, CO] = 
    
    getOutputStrategy(unknownOutputSize, dc_linear_buffer(this.allocVal)) match {
     case OutputBuffer => DeliteCollectBufferOutput[O, I, CO](
        eV = this.eV,
        sV = this.sV,
        iV = this.iV,
        iV2 = this.iV2,
        allocVal = this.allocVal,
        aV2 = this.aV2,
        alloc = reifyEffects(this.alloc(sV)),
        update = reifyEffects(dc_update(allocVal,v,eV)),
        append = reifyEffects(dc_append(allocVal,v,eV)),
        appendable = reifyEffects(dc_appendable(allocVal,v,eV)),
        setSize = reifyEffects(dc_set_logical_size(allocVal,sV)),
        allocRaw = reifyEffects(dc_alloc[O,I](allocVal,sV)),
        copyRaw = reifyEffects(dc_copy(aV2,iV,allocVal,iV2,sV)),
        finalizer = reifyEffects(this.finalizer(allocVal))
      )

      case OutputFlat => DeliteCollectFlatOutput[O, I, CO](
        eV = this.eV,
        sV = this.sV,
        allocVal = this.allocVal,
        alloc = reifyEffects(this.alloc(sV)),
        update = reifyEffects(dc_update(allocVal,v,eV)),
        finalizer = reifyEffects(this.finalizer(allocVal))
      )
    }

    // loop elem
    lazy val body: Def[CO] = DeliteCollectElem[O,I,CO](
      buf = this.buf,
      iFunc = reifyEffects(this.iFunc),
      unknownOutputSize = this.unknownOutputSize,
      numDynamicChunks = this.numDynamicChunks,
      eF = this.eF,
      iF = this.iF,
      sF = reifyEffects(dc_size(eF)),
      aF = reifyEffects(dc_apply(eF,iF))
    )
  }
```