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

## DeliteLoops

![Alt text](http://g.gravizo.com/g?
  digraph G {
    rankdir=BT;
    Def [shape=box];
    ;
    AbstractLoop [shape=box];
    AbstractLoop -> Def;
    ;
    DeliteOp [shape=box];
    DeliteOp -> Def;
    ;
    DeliteOpLoop [shape=box];
    DeliteOpLoop -> AbstractLoop;
    DeliteOpLoop -> DeliteOp;
    ;
    DeliteOpCollectLoop [shape=box];
    DeliteOpCollectLoop -> DeliteOpLoop;
    ;
    DeliteOpFlatMapLike [shape=box];
    DeliteOpFlatMapLike -> DeliteOpCollectLoop;
    ;
    DeliteOpFoldLike [shape=box];
    DeliteOpFoldLike-> DeliteOpCollectLoop;
    ;
    DeliteOpReduceLike [shape=box];
    DeliteOpReduceLike -> DeliteOpCollectLoop;    
  }
)

## DeliteElems
![Alt text](http://g.gravizo.com/g?
  digraph G {
    rankdir=BT;
    Def [shape=box];
    ;
    DeliteLoopElem [shape=box];
    ;
    DeliteHashElem [shape=box];
    DeliteHashElem -> Def;
    ;
    DeliteHashIndexElem [shape=box];
    DeliteHashIndexElem -> DeliteHashElem;
    DeliteHashIndexElem -> DeliteLoopElem;
    ;
    DeliteCollectBaseElem [shape=box];
    DeliteCollectBaseElem -> Def;
    DeliteCollectBaseElem -> DeliteLoopElem;
    ;
    DeliteFoldElem [shape=box];
    DeliteFoldElem -> DeliteCollectBaseElem;
    ;
    DeliteReduceElem [shape=box];
    DeliteReduceElem -> DeliteCollectBaseElem;
    ;
    DeliteCollectElem [shape=box];
    DeliteCollectElem -> DeliteCollectBaseElem;
    ;
    DeliteHashReduceElem [shape=box];
    DeliteHashReduceElem -> DeliteHashElem;
    DeliteHashReduceElem -> DeliteLoopElem;
    ;
    DeliteHashCollectElem [shape=box];
    DeliteHashCollectElem -> DeliteHashElem;
    DeliteHashCollectElem -> DeliteLoopElem;
    ;
    DeliteForeachElem [shape=box];
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
final lazy val field: Type = copyTransformedOrElse(_.field)(someValue).asInstanceOf[Type]
```

It means that the field's value is `someValue` which is either a default value or another field in the class. And the reason why it is written this way is to automatically apply a transformer to all members of a class. 

For consiceness reasons, for now on, I will rewrite the above as :

```scala
val field: Type = someValue
```

## DeliteOpLoop
```scala
class DeliteOpLoop extends AbstractLoop[A] with DeliteOp[A] {
    type A:Manifest

    val size: Exp[Int] // Size of the loop
    val v: Sym[Int] = fresh[Int] // Symbol for index inside the body
    val body: Def[A] // Expression representing the body of the loop
    val numDynamicChunks: Int = 0 // Parallellism of the loop
}
```
Base of all the delite parallel loops. 

## DeliteOpCollectLoop
```scala
class DeliteOpCollectLoop[O:Manifest, R:Manifest] extends DeliteOpLoop[R] {
    val size: Exp[Int] // Size of the loop
    val v: Sym[Int] = fresh[Int] // Symbol for index inside the body
    val body: Def[A] // Expression representing the body of the loop
    val numDynamicChunks: Int = 0 // Parallellism of the loop

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
val numDynamicChunks: Int
```

Base class for all of the loop bodies of delite.

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
    lazy val body: Def[CO] = copyBodyOrElse(DeliteCollectElem[O,I,CO](
      buf = this.buf,
      iFunc = reifyEffects(this.iFunc),
      unknownOutputSize = this.unknownOutputSize,
      numDynamicChunks = this.numDynamicChunks,
      eF = this.eF,
      iF = this.iF,
      sF = reifyEffects(dc_size(eF)),
      aF = reifyEffects(dc_apply(eF,iF))
    ))
  }
```

### TODO:
### What is the difference between DeliteOps.scala and DeliteOpsIR.scala
### Is there a better way to mirror than original ?
### 