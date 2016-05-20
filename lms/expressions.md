# Base & Expressions
| Type | Explanation |
| ---- | ----------- |
| `Exp[+T]` | Reptresents an ast node that will return an expression of type T. Contains type information. |
| `Const[+T](x: T)`<br>`extends Exp[T]` | Represents a constant expression of type T |
| `Sym[+T](id: Int)`<br>`extends Exp[T]` | Represents a symbol that is associated to a Def[T]. Carries source context information for debugging. May be associated to several source positions due to CSE. |
| `Def[+T]` | IR node that can be defined by a DSL or by LMS (eg. IfThenElse, DeliteOpForeach)  |
| `Stm` | Abstract TP |
| `TP[+T](`<br>`  sym: Sym[T],`<br> `rhs: Def[T]`<br>`) extends Stm` | Typed pair. Links one definition to a symbol |
| `TTP(`<br>`val lhs: List[Sym[Any]],`<br>`val mhs: List[Def[Any]],`<br>`val rhs: FatDef`<br>`) extends Stm` | Fat version of a typed pair. Can define multiple symbols with a fat definition (FatDef) |
| `Variable[+T](`<br>`e: Exp[Variable[T]]`<br>`)` | Used to encapsulate a variable reference. Use of a variable will trigger generation of code to read the variable. (see ReadVarImplicits) |




| Method | Explanation |
| ------ | ----------- |
| `object Def { `<br>`def unapply[T](e: Exp[T]): Option[Def[T]]`<br>`}` | Allows the use of pattern matching to find def node that corresponds to a particular symbol |
| `implicit toAtom(d: Def[T]): Exp[T]` | Creates symbol for definition, either by creating a new symbol or returning the symbol of an equal definition that already exists |
| | |


### How is the IR represented
The IR is represented as a sea of nodes. Similar to SSA form, it allows common optimizations like CSE and DCE to be perfomed.


### How does CSE (Common subexpression elimination) happen ?
When the dsl is executed and the nodes are formed, every node extends Def, but they can only reference symbols or constants (Exp). The toAtom is thus called everytime a node is created, and it takes care of looking up if this node already exists or not. It also registers the node if it is the first time it is seen.


### How does DCE (Dead code elimination) happen
Whenever 


### How do I translate from `Def`s to `Exp`s and back ?
Going from a `Def` to an `Exp` basically implies 

### What is mirror ?
Mirroring is like evaluating the IR, 

### What is reflect ?


### What is reify ?


### Arrays ? How do they work ?




