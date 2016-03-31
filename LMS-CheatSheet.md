## LMS / Delite Wiki
| Type | Explanation |
| ---- | ----------- |
| `Exp[+T]` | Reptresents an ast node that will return an expression of type T. Contains type information. |
| `Const[+T](x: T)`<br>`extends Exp[T]` | Represents a constant expression of type T |
| `Sym[+T](id: Int)`<br>`extends Exp[T]` | Represents a symbol that is associated to a Def[T]. Carries source context information for debugging. May be associated to several source positions due to CSE. |
| `Variable[+T](`<br>`e: Exp[Variable[T]]`<br>`)` | ??? Not sure ??? |
| `Def[+T]` | IR node that can be defined by a DSL or by LMS (eg. IfThenElse, DeliteOpForeach)  |
| `Stm` | Abstract TP |
| `TP[+T](`<br>`  sym: Sym[T],`<br> `rhs: Def[T]`<br>`) extends Stm` | Typed pair. Links one definition to a symbol |
| `TTP(`<br>`val lhs: List[Sym[Any]],`<br>`val mhs: List[Def[Any]],`<br>`val rhs: FatDef`<br>`) extends Stm` | Fat version of a typed pair. Can define multiple symbols with a fat definition (FatDef) |



| Method | Explanation |
| ------ | ----------- |
| `toAtom(d: Def[T]): Exp[T]` | Creates symbol for definition, either by creating a new symbol or returning the symbol of an equal definition that already exists (in scope?) (reflect effects?) |
|  |  |

## Assumptions
**From Vera (page 5)**: Definitions can only refer to Expressions (Def[T] can only contain Exp[T] -> Symbols or Constants), but what about nested scopes on page 7 ?

## FAQ

### OverloadHack, WTF?
def m(r: Rep[Int]) cannot be differentiated from def m(r: Rep[String]) because of type erasure, adding an implicit argument of type Overload1 and Overload2 allows to differentiate. ??? How is it used though ???

### What does the parent of a SourceContext represent ?
Source context is basically a stack, each new function is a new source context with the englobing method as parent.

### How to print Delite IR
Use the IRPrinter in the metadata branch

### I don't see any constant folding transformer
Constant folding is handled by rewrite rules. For example in MathOps.mirror, call constructor function that will then in turn apply constant folding. (if there is no rewrite rule though, there is opportunity for constant folding)

### Why are the signatures of the functions in EmbeddedControls so strange (call by name/ call by value)?
It's a bug but it is not very important since scala-virtualize is never actually going to use them. Either there is something that overloads/overrides the functions, or the call stays non-virtual (i.e. if(...) { ... } else { ... }) 

### What does splicing mean ?
Join or insert. Ex. Splicing TypeTags to construct TypeTags for Composite types

### What does homoiconicity mean ?
*From wikipedia:* In computer programming, homoiconicity (from the Greek words homo meaning the same and icon meaning representation) is a property of some programming languages in which the program structure is similar to its syntax, and therefore the program's internal representation can be inferred by reading the text's layout.[1] If a language is homoiconic, it means that the language text has the same structure as its abstract syntax tree (i.e. the AST and the syntax are isomorphic). This allows all code in the language to be accessed and transformed as data, using the same representation.

### What does Reify mean?
To make real, Ex. T -> Expr[T]

### Stm.lhs implemented as infix_lhs, why ?
So Stm is extensible, eg. TPP has a different definition of lhs than TP.

### What is StencilAnalysis 
Not sure. something to do with blocking ????

### DeliteLoopElem does not extend Def even though all of the subclasses extend or mixin Def or DeliteHashElem which in turn extends Def
???

### What is OpType useful for in DeliteOp
???

### Why does GraphTraversal not really traverse anything
traverseStatement is defined in BlockTraversal..... 

### Where is the asbtract type Block[+T] from BlockTraversal actually defined in codegens ?
???

### TupleGenBase extends GenericCodeGen even though BaseGenStruct extends GenericNestedCodegen which in turn extends GenericCodeGen

### What exactly do the functions reifyEffects and reflectEffects do ?
```scala
def reifyEffects[A:Manifest](block: => Exp[A]): Block[A]
def reifyEffectsHere[A:Manifest](block: => Exp[A]): Block[A]

def reflectMutableSym[A](s: Sym[A]): Sym[A]
def reflectMutable[A:Manifest](d: Def[A])
def reflectWrite[A:Manifest](write0: Exp[Any]*)(d: Def[A])
def reflectEffect[A:Manifest](x: Def[A])
def reflectEffect[A:Manifest](d: Def[A], u: Summary)
```

### How can I get loopfusion to work on my nodes
????

### mirror vs mirrorDef
????

### When are tranformers run exactly ?
Manually in LMS, delite has something to chain them (???)

### Debugger to visualize performance aspects
????

### What is the advantage of a forward preserving transformer ?



## Classes
```scala
// Simple exp transformer with no context (effects?)
trait AbstractTransformer {	
	/* Transform list of symbols and collect only the 
	 * ones that end up as 	Sym[A] after transform.
	 */
	def onlySyms[A](xs: List[Sym[A]]): List[Sym[A]]
}
```

```scala
// Transforms Exp’s and Block’s to other Exp’s and Block’s
// Still no context
trait AbstractSubstTransformer extends AbstractTransformer {
	/* withSubstScope saves the subst map and then allows
	 * the code passed in block to mutate it, to emulate a 
	 * sort of stack for subst.
	 */
	var subst: Map[Exp[Any], Exp[Any]]
	def withSubstScope[A](block: => A): A
}
```

```scala
trait Scheduling {
	/* Creates a map from every symbol to it’s Statement 
	 *	and positionin the list of statements
	 */
	def buildScopeIndex(scope: List[Stm]): 
			IdentityHashMap[Sym[Any], (Stm,Int)]
	
	/* For all symbols in syms, find the statement where it is 
	 * bound and order the resulting list according to original
	 * order in which in was defined.
	 */
	def scheduleDepsWithIndex(syms: List[Sym[Any]], 
			cache: IdentityHashMap[Sym[Any], (Stm,Int)]): 
			List[Stm]
			
	/* Starts with the result statement and extracts it’s 
	 * dependecies recursively until all of the reachable 
	 * statements are in the schedule. If there is a cyclic 
	 * dependency, the schedule is recursive (recursive function?) 
	 * and a warning is emitted. The sort parameter will ignore 
	 * the warnings if set to false.
	 */
	def getSchedule(scope: List[Stm])
		(result: Any, sort: Boolean = true): List[Stm]
	
	/* Changes the syms function with symsFreq to use hot/cold  
	 * heuristic for code motion (M for motion?).
	 */	
	def getScheduleM(scope: List[Stm])
			(result: Any, cold: Boolean, hot: Boolean): List[Stm]
	/* From comment:
	 * for each symbol s in sts, find all statements that 
	 * depend on it.
	 */
	def getFatDependentStuff(scope: List[Stm])
			(sts: List[Sym[Any]]): List[Stm]
}
```

```scala
object GraphUtil {
	/* Strongly connected components in topological order from 
	 * directed graph. If a component has more 
	 * than one element, it means there must be a cycle in it.
	 */
	def stronglyConnectedComponents[T](start: List[T], 
			succ: T=>List[T]): List[List[T]]
}
```

```scala
// From comment:
// traversals are stateful (scheduling is stateless)
trait GraphTraversal extends Scheduling {
	// globalDefs are defined in Expressions with comment: 
	// Graph traversal state
	def availableDefs: List[Stm] = globalDefs
	
	/*
	 * This class basically just calls getSchedule and 
	 * getFatDependentStuff from it's parent, only with 
	 * availableDefs as scope. And availableDefs will reflect 
	 * the state in which the globalDefs in the Expressions mixin
	 * are in.
	 */
}
```
```scala
trait CodeMotion extends Scheduling {
	// ???
	def getExactScope[A](currentScope: List[Stm])
			(result: List[Exp[Any]]): List[Stm] = {
		
		// val y = loop(10){ i -> x + 5 + 2*i }
		// Sym(0) -> x
		// Sym(1) -> i
		// Sym(2) -> x + 5
		// Sym(3) -> 2*i
		// Sym(4) -> x + 5 + 2*i
		// Sym(5) -> result of the loop
		// A - TP(Sym(2),IntPlus(Sym(0),Const(5)))
		// B - TP(Sym(5),Loop(Const(10),Sym(1),Block(Sym(4))))
		// C - TP(Sym(3),IntTimes(Const(2),Sym(1)))
		// D - TP(Sym(4),IntPlus(Sym(2),Sym(3)))

		//
		// e1 -> statements in current scope (A, B, C, D)
		
		// bound -> symbols that are inside loops (1)
		
		// g1 -> statements that have i in their 
		// transitive dependencies (C, D)
		
		// h1 -> statements that do not depend on 
		// a bound symbol (A, B)
		
		// g1.flatMap { t => syms(t.rhs) } -> (1, 2, 3)
		// flatMap(s => h1 filter(_.lhs contains s)) -> (A, C)
		// all statements one step away from g1
		
		// ??? need to understand what getScheduleM 
		// does in more details
}
```
```scala
trait NestedGraphTraversal extends GraphTraversal 
		with CodeMotion {
	// From comment : no, it's not a typo
	// Why is it not a typo ?	
	var innerScope: List[Stm] = null 
	
	// Start with globalDefs is innerScope is not set, 
	// else innerScope
	override def availableDefs: List[Stm]
	
	// Save old scope, change mutable innerScope for scope,
	// call body, and then restore old scope before returning.
	def withInnerScope[A](scope: List[Stm])(body: => A): A
	
	// Define the scope as being all the dependencies (schedule)
	// for the expressions in result to call body.
	def focusSubGraph[A](result: List[Exp[Any]])(body: => A): A
	
	// From comment: strong order for levelScope (as obtained by 
	// code motion), taking care of recursive dependencies.
	// results later refered as (levelScope2, recursive)
	def getStronglySortedSchedule2(
			scope: List[Stm], 
			level: List[Stm], // ??? result of getExactScope
			result: Any
		): (List[Stm], List[Sym[Any]])
	
	// Same as focusSubgraph, with the exception of 
	// whatever levelScope is. Inner scope is last 
	// innerScope - levelScope, and levelScope is passed 
	// as a parameter to block.
	def focusExactScopeSubGraph[A](result: List[Exp[Any]])
			(body: List[Stm] => A): A
	
	
}
```
```scala
// Defines abstract type Block[+T] + methods on it
trait BlockTraversal extends GraphTraversal {
	def getFreeVarBlock(start: Block[Any], local: List[Sym[Any]]): List[Sym[Any]]
	def getFreeDataBlock[A](start: Block[A]): List[(Sym[Any],Any)]
	
	def getBlockResult[A](s: Block[A]): Exp[A]
	def getBlockResultFull[A](s: Block[A]): Exp[A]
	
	def traverseBlock[A](block: Block[A]): Unit
	def traverseStm(stm: Stm): Unit
	
	def reset() ????? wtf
}
```

```scala
// Implements methods defined in BlockTraversal with the scope
// management defined in NestedGraphTraversal
trait NestedBlockTraversal extends BlockTraversal 
		with NestedGraphTraversal {
	
	def traverseStmsInBlock[A](stms: List[Stm]): Unit
}
```

```scala
// Substitution transformer + scoping handled by FatBlocktraversal
// Uses only past statements (and current) to transform statement
trait ForwardTransformer extends internal.AbstractSubstTransformer
		with internal.FatBlockTraversal {
	
	override def traverseStm(stm: Stm): Unit = {
		// Try to apply substitution
		// not subst -> 
		//		call transform on stm and add to substitution
		// else 
		//		must be a recursive symbol (why?)
		//		so transform but do not add to substitution
	}
	
	// Pretty much call mirror on the statement
	def transformStm(stm: Stm): Exp[Any]
	
	// ???
	override def reflectBlock[A](block: Block[A])
}
	
```


## Ideas
 - Use scala.Dynamic to replace infix_ and thus implicits in new lms macro-virt
 - Use forge to statically resolve the linearization of mixins so the compiler doesn't need to do it when compiling a dsl (can you do it while preserving source compat ?)
 - Find a way to migrate bound syms at the end of a list of commutative operations, so code motion can kick in
 - Track down places where SourceContexts are missing
 

## HS Questions
 - How to check for implicit resolution bottlenecks?

## Tasks
Transformer to code cpp => string concat

