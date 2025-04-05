+++
title = "rustc浅析: Unsafety Checking"
date = "2024-12-15"
taxonomies.tags = [
    "Rust",
    "Compiler",
    "Chinese",
]
+++

**这一步检查的实现位于[check_unsafety.rs](https://github.com/rust-lang/rust/blob/master/compiler/rustc_mir_build/src/check_unsafety.rs)。**

## 引入

Rust中，所有无法由类型系统证明正确性的代码（如裸指针的解引用），以及被`unsafe`修饰的`traits`，`functions`和`methods`，都被视为是unsafe的。这一部分的操作需要被放入`unsafe`块内。Unsafety Checking确保unsafe操作不被用于`unsafe`块之外。



## THIR

```shell
-----------
Source Code
-----------
     |
-----------
 Rust AST
-----------
     |
-----------
    HIR
-----------
     |
-----------
   THIR
-----------
     |
-----------
    MIR     <-- Control Flow Graph
-----------
```

Unsafety Checking基于THIR，理由如下：

1. 和HIR相比，使用THIR需要考虑的情况更少。比如`unsafe function calls`和`unsafe method calls`在THIR中的表示是相同的。
2. 不用MIR是因为Unsafety Checking不需要控制流，并且对于一些表达式，MIR没有像THIR那么精确的span。



## 辨认unsafe操作

绝大部分的unsafe操作可以通过检查THIR中的`ExprKind`和检查参数类型来辨认。比如说想要辨认裸指针解引用的操作，可以通过检查具有裸指针参数的 `ExprKind::Deref`：

`UnsafeVisitor::visit_expr()`:

```rust
// https://github.com/rust-lang/rust/blob/master/compiler/rustc_mir_build/src/check_unsafety.rs#L538-L553
ExprKind::Deref { arg } => {
    if let ExprKind::StaticRef { def_id, .. } | ExprKind::ThreadLocalRef(def_id) =
        self.thir[arg].kind
    {
        if self.tcx.is_mutable_static(def_id) {
            self.requires_unsafe(expr.span, UseOfMutableStatic);
        } else if self.tcx.is_foreign_item(def_id) {
            match self.tcx.def_kind(def_id) {
                DefKind::Static { safety: hir::Safety::Safe, .. } => {}
                _ => self.requires_unsafe(expr.span, UseOfExternStatic),
            }
        }
    } else if self.thir[arg].ty.is_unsafe_ptr() {
        self.requires_unsafe(expr.span, DerefOfRawPointer);
    }
}
```



### Access fields of a `union`

考虑以下代码：

```rust
#[repr(C)]
union MyUnion {
    f1: i32,
    f2: f32,
}

fn foo(u: MyUnion) {
  u.f1 = 6;
  let f = unsafe { u.f1 };
}
```

- 会发现，写入`u.f1`是safe的，而读取`u.f1`是unsafe的。

因此检查器会允许union fields直接出现在赋值表达式的LHS，而其它情况全部报错。

同时读取union fields的操作也会发生在模式匹配，因此那里也需要被检查器走一遍：

`UnsafeVisitor::visit_expr()`:

```rust
// https://github.com/rust-lang/rust/blob/master/compiler/rustc_mir_build/src/check_unsafety.rs#L623-L643
ExprKind::Field { lhs, variant_index, name } => {
    let lhs = &self.thir[lhs];
    if let ty::Adt(adt_def, _) = lhs.ty.kind() {
    if adt_def.variant(variant_index).fields[name].safety == Safety::Unsafe {
        self.requires_unsafe(expr.span, UseOfUnsafeField);
        } else if adt_def.is_union() {
            if let Some(assigned_ty) = self.assignment_info {
                if assigned_ty.needs_drop(self.tcx, self.typing_env) {
                    // This would be unsafe, but should be outright impossible since we
                    // reject such unions.
                    assert!(
                        self.tcx.dcx().has_errors().is_some(),
                        "union fields that need dropping should be impossible: {assigned_ty}"
                    );
                }
            } else {
                self.requires_unsafe(expr.span, AccessToUnionField);
            }
        }
    }
}
```

`UnsafeVisitor::visit_pat()`:

```rust
// https://github.com/rust-lang/rust/blob/master/compiler/rustc_mir_build/src/check_unsafety.rs#L341-L366
match &pat.kind {
    PatKind::Leaf { subpatterns, .. } => {
        if let ty::Adt(adt_def, ..) = pat.ty.kind() {
            for pat in subpatterns {
                if adt_def.non_enum_variant().fields[pat.field].safety == Safety::Unsafe {
                    self.requires_unsafe(pat.pattern.span, UseOfUnsafeField);
                }
            }
            if adt_def.is_union() {
                let old_in_union_destructure =
                    std::mem::replace(&mut self.in_union_destructure, true);
                visit::walk_pat(self, pat);
                self.in_union_destructure = old_in_union_destructure;
            } else if (Bound::Unbounded, Bound::Unbounded)
                != self.tcx.layout_scalar_valid_range(adt_def.did())
            {
                let old_inside_adt = std::mem::replace(&mut self.inside_adt, true);
                visit::walk_pat(self, pat);
                self.inside_adt = old_inside_adt;
            } else {
                visit::walk_pat(self, pat);
            }
        } else {
            visit::walk_pat(self, pat);
        }
    }
    // ...
}
```



### Anonymous Constant

HIR用`AnonConst`表示 *anonymous constant（匿名常量）*。

```rust
// https://github.com/rust-lang/rust/blob/master/compiler/rustc_hir/src/hir.rs#L1682-L1696
 /// A constant (expression) that's not an item or associated item, 
 /// but needs its own `DefId` for type-checking, const-eval, etc. 
 /// These are usually found nested inside types (e.g., array lengths) 
 /// or expressions (e.g., repeat counts), and also used to define 
 /// explicit discriminant values for enum variants. 
 /// 
 /// You can check if this anon const is a default in a const param 
 /// `const N: usize = { ... }` with `tcx.hir().opt_const_param_default_param_def_id(..)` 
 #[derive(Copy, Clone, Debug, HashStable_Generic)] 
 pub struct AnonConst { 
     pub hir_id: HirId, 
     pub def_id: LocalDefId, 
     pub body: BodyId, 
     pub span: Span, 
 } 
```

- 这里表明到，*anonymous constant* 并不是一个item。

但实际上，`AnonConst`在许多地方会被当作成一个item来处理，比如本文讲到的Unsafety Checking。*anonymous constant*并不具备继承unsafety的能力，考虑以下代码：

```rust
const unsafe fn foo() -> usize { 1 }

fn main() {
    unsafe {
        let _x = [0; foo()];
    }
}
```

有如下报错信息：

```shell
Compiling playground v0.0.1 (/playground)
warning: unnecessary `unsafe` block
 --> src/main.rs:4:5
  |
4 |     unsafe {
  |     ^^^^^^ unnecessary `unsafe` block
  |
  = note: `#[warn(unused_unsafe)]` on by default

error[E0133]: call to unsafe function `foo` is unsafe and requires unsafe function or block
 --> src/main.rs:5:22
  |
4 |     unsafe {
  |     ------ items do not inherit unsafety from separate enclosing items
5 |         let _x = [0; foo()];
  |                      ^^^^^ call to unsafe function
  |
  = note: consult the function's documentation for information on how to avoid undefined behavior

For more information about this error, try `rustc --explain E0133`.
warning: `playground` (bin "playground") generated 1 warning
error: could not compile `playground` (bin "playground") due to 1 previous error; 1 warning emitted
```

因为这里`foo()`被表示为*anonymous constant*，而*anonymous constant*在Unsafety Checking中被当作成一个item来处理，因此其不具备继承unsafety的能力，因此报错。



## 参考

1. [Rustc Dev Guide: Unsafety Checking](https://rustc-dev-guide.rust-lang.org/unsafety-checking.htm)
2. [Issue #133441: An unsafe const fn being used to compute an array length or const generic is incorrectly described as being an "item".](https://github.com/rust-lang/rust/issues/133441#issuecomment-2532923245)