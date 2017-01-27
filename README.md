# Orthogonal Classes
Proposed EcmaScript Class Syntax clarifying orthogonal concerns

## Non-Orthogonal Goals
  1. Compatible with existing EcmaScript classes
  1. Express private fields and public properties
  1. Easy to understand
  1. Hard to misunderstand
  1. Initialization syntax that does not invite confusion with assignment
  1. Enable an annotation to annotate many fields and/or properties together
  1. Express private methods and private static methods
  1. Be understandable in a "shallow wide" sense
  1. Invite understanding to become "deeper and narrower"
  1. Don't cut off sensible choices without good reason
  1. Avoid combinatorial explosion of ad hoc syntaxes to express many sensible choices
  1. Preserve investment in existing proposals -- stay similar to them
  
## Orthogonal Dimensions of Declarative Class State
  1. Placement: Is the state on class/constructor vs prototype vs instance?
  1. Visibility: Public property vs private field
  1. Kind: What value is this state initialized to?
  
## Proposal

Starting with [Grammar Summary, Functions and 
Classes](https://www.ecma-international.org/ecma-262/7.0/#sec-functions-and-classes):

```
ClassElement :
  Annotation? ElementPlacement ElementVisibility ElementKind
  
ElementPlacement :
  "static" | Empty | "own"
  
Empty :

ElementVisibility :
  Empty | "#"
  
ElementKind :
  MethodDefinition | BindingList ";"
```

This syntax directly expresses the desired orthogonality. However, in the orthogonal matrix, there are some cases that, if allowed, would violate some of our goals. So the actual proposal sacrifices orthogonality in banning these cases. The above grammar is therefore a cover grammer where the rejection is the post-parse check. The rejected cases are

```
Empty ElementVisibility BindingList ";"
ElementPlacement # get PropertyName "(" ")" "{" FunctionBody "}"
ElementPlacement # set PropertyName "(" PropertySetParameterList ")" "{" FunctionBody "}"
```
Aside from this rejection, the rest of the proposal preserves the remaining orthogonality.


### Placement

#### static

If the initial keyword is **`static`** then the declaration initializes state on  the class/constructor. This is compatible with existing proposed uses of **`static`**.

#### own

If the initial keyword is **`own`** then the declaration initializes state on the instance. This differs from current proposals for initializing state on the instance. Besides the general benefits of orthogonality, using **`own`** for this has specific advantages over these proposals without **`own`**:
  1. The syntax `x = 9;` looks like an assignment and will be misunderstood as an assignment. The syntax `own x = 9;` looks like a variable declaration and initialization, making DefineProperty semantics less surprising.
  1. Grouping multiple declarations enables them to be annotated together. However, several people said they find the syntax `x = 9, y, z = 10;` confusing in this position. The syntax `own x = 9, y, z = 10;` is not surprising, again, because it follows the existing syntactic pattern of variable declarations and initializations.

By orthogonality `own MethodDefinition` defines a method on the instance. This might rarely be useful, but banning it would be surprising.

#### Neither static nor own

In the absence of either the **`static`** or **`own`** initial keywords, the declaration initializes state on the prototype. When coupled with MethodDefinition, this is compatible with existing syntax. 

Coupled with the BindingList form, this would recreate the confusions we just enumerated above with `x = 9, y, z = 10;` together with the further confusion that this would be initialized on the prototype. Instead, we statically reject this one case. For those writing code, this violation of orthogonality can be a rude surprise. But for those reading code which is not statically rejected, there is no such violation or rude surprise.

### Visibility

The presence of the **`#`** sigil indicates a private field. The existing private state proposal uses this specifically for a private instance field, which we would write `own # ...`. The existing proposal postpones the issue of how one would express private methods or private static methods. Here, those other cases simply falls out. `# MethodDefinition` is a private method definition, i.e., the method is the initial value of a private field of that name on the prototype. `static # MethodDefinition` defines a private static method, i.e., the method is the initial value of a private field on the class/constructor.

In the WeakMap-like way of defining private state, these additional cases are specified the same way: the private names are in scope over the same body of code. But rather than using instances as the keys of the weakmap-like collection named by those names, the key would be either the prototype or the class/constructor. Of course, implementations can implement by any means that is not observably different from that.

### Kind -- MethodDefinition or BindingList

The MethodDefinition can define a normal method, a generator method, an accessor, or the constructor. Accessor properties are not about the initial value of the property, but rather about the nature of the property itself. This only makes sense for public properties. A private accessor field is not meaningful.

The normal constructor declaration serves two purposes: To define the behavior of the class/constructor object and to initialize the `"constructor"` property of the prototype to point at this class/constructor. If treated orthogonally, we would continue to have a MethodDefinition for the name `"constructor"` define the behavior of the class/constructor itself. However, what objects are initialized to point at this constructor would be determined orthogonally. Again most of these choices may rarely be useful but there is no reason to violate orthogonality to surprising disallow them. In addition, one case is hugely useful:

Classes will often be defined where the authority provided by holding an instance of a class should not necessarily confer the authority to invoke the constructor and make new instances. This came up most recently with WeakRefs but naturally occurs in many places. I have written much Java code whose correctness depends on the instances providing less power than their class object provides. In this design, if the `"constructor"` method definition is made static, then it is initialized to point at itself. If made private, no matter where it is placed, only code inside the class can follow this pointer back to the class/constructor. Clients of the instances have no such access and so cannot so navigate.