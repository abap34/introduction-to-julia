---
marp: true
paginate: true
math: mathjax
theme: honwaka
---

<style>
{
  font-size: 0.8rem;
}
</style>

<!-- _class: lead -->

# Julia コンパイラの実装

<br>


1. Julia の実行プロセス
2. IR と最適化
   1. パース
   2. lowering
   3. 型推論
   4. SSA形式の変換と最適化
   5. LLVM IR と コード生成


---

<!-- _header: Julia の実行プロセス -->


![center 90%](img/image-3.png)


---

<!-- _header: Julia の実行プロセス -->

![bg right 90%](img/image-3.png)


1. 独自の IR で型推論や最適化
2. SSA 形式に直したものにさらに最適化
3. LLVM IR に変換して LLVM の最適化パスを通す
4. コード生成


---

<!-- _header: Julia の 実行プロセス -->

<!-- 左 30% 右 70% -->
<div style="display: flex; flex-direction: row; align-items: stretch; gap: 2em;">

<!--　上下で中央よせ -->
<div style="flex: 0 0 30%; display: flex; flex-direction: column; justify-content: center; text-align: left !important;">

## 注意

- トップレベルの実行
- 簡単なヒューリスティック

によってコンパイル不要と判断された場合はインタプリタが実行を担う

</div>

<div style="flex: 0 0 66%;">


```c
JL_DLLEXPORT jl_value_t *jl_toplevel_eval_flex(jl_module_t *JL_NONNULL m, jl_value_t *e, int fast,
int expanded,  const char **toplevel_filename, int *toplevel_lineno)
{
  ...
  else if (head == jl_global_sym) {
      size_t i, l = jl_array_nrows(ex->args);
      for (i = 0; i < l; i++) {
          jl_value_t *arg = jl_exprarg(ex, i);
          jl_declare_global(m, arg, NULL, 0);
      }
      JL_GC_POP();
      return jl_nothing;
  }
  ...
  int has_ccall = 0, has_defs = 0, has_loops = 0, has_opaque = 0, forced_compile = 0;
  ...
  body_attributes((jl_array_t*)thk->code, &has_ccall, &has_defs, &has_loops,
  &has_opaque, &forced_compile);

  jl_value_t *result;
  if (has_ccall || ((forced_compile || (!has_defs && fast && has_loops)) && ...)) {
      // use codegen
      mfunc = jl_method_instance_for_thunk(thk, m);
      ...
      if (!has_defs && jl_get_module_infer(m) != 0) {
          (void)jl_type_infer(mfunc, world, SOURCE_MODE_ABI, jl_options.trim);
      }
      result = jl_invoke(/*func*/NULL, /*args*/NULL, /*nargs*/0, mfunc);
      ct->world_age = last_age;
  }
  else {
      // use interpreter
      ...
      result = jl_interpret_toplevel_thunk(m, thk);
      ct->world_age = last_age;
  }

  JL_GC_POP();
  return result;
}
```

<div class="center">


`toplevel.c#jl_toplevel_eval_flex`


</div>

</div>


</div>

---

<!-- _header: Julia の 実行プロセス -->

**⚠️ 注意**
トップレベル実行 / 簡単なヒューリスティックでコンパイル不要と判断された場合は
基本的に AST を直接実行しているだけなので今回はあまり触れず，
コンパイルされる場合にフォーカスします

<span class="gray">


(小噺:
一方でこのことはトップレベルでの実行を abstract interpret することの
難しさにつながっていてとても大変 😢)


</span>


---


<!-- _header: Julia のコンパイルプロセス -->

**📝 Julia の IR とその変化をみながら Julia のコンパイルプロセスを追っていきます**

- `Expr`
- `CodeInfo`
- `IRCode`
- LLVM IR


---

<!-- _header: パース -->

以前は Julia コンパイラのフロントエンドはほぼ Scheme (FemtoLisp) で書かれていたが
多くのユーザにとっては使いづらいしメンテナンスコストが高い 😰

##  → Julia による実装に置き換えられつつある 😆 

:new: 1.10 からのデフォルトパーサ: [JuliaLang/JuliaSyntax.jl](https://github.com/JuliaLang/JuliaSyntax.jl)

- 手書きの再帰下降
- ロスレス
- 正確な provenance は LS などの開発で非常に重要なので現代では重視されておる
  - 類似の話: [Prism⁠⁠：エラートレラントな⁠⁠、まったく新しいRubyパーサ](https://gihyo.jp/article/2024/01/ruby3.3-prism)


---

<!-- _header: Expr -->

<div class="columns">

<div>

**基本的にコンパイラ内部で**
**使われている AST のデータ構造:** <span style="font-size: 1.4em;">`Expr`</span>
... といっても単に `head` と `args` があるだけのよくあるやつ


</div>

```julia
julia> Meta.parse("""
       for i in 1:10
           @show i
       end
       """) |> dump
Expr
  head: Symbol for
  args: Array{Any}((2,))
    1: Expr
      head: Symbol =
      args: Array{Any}((2,))
        1: Symbol i
        2: Expr
          head: Symbol call
          args: Array{Any}((3,))
            1: Symbol :
            2: Int64 1
            3: Int64 10
    2: Expr
      head: Symbol block
      args: Array{Any}((3,))
        1: LineNumberNode
          line: Int64 2
          file: Symbol none
        2: Expr
          head: Symbol macrocall
          args: Array{Any}((3,))
            1: Symbol @show
            2: LineNumberNode
              line: Int64 2
              file: Symbol none
            3: Symbol i
        3: LineNumberNode
          line: Int64 3
          file: Symbol none
```

</div>



---

<!-- _header: lowering -->

## `CodeInfo`

AST に対して:

1. macro の展開
2. desugar
3. クロージャの変換

<div style="position: absolute; top: 200px; left: 420px; width: 600px; border: 1px solid #333; padding: 20px; background: #f9f9f9; font-size: 0.80em;">

この処理に必要な

- Racket を参考にしたマクロ展開のアルゴリズム
- スコープ解析
- 大量の desugar の規則 😇

などが実装されている

</div>


<svg style="position: absolute; top: 200px; left: 100px; width: 400px; height: 300px;">
  <line x2="300" y2="150" x1="240" y1="160"
  style="stroke:#333; stroke-width:2" marker-end="url(#arrowhead)"/>
  <defs>
    <marker id="arrowhead" markerWidth="10" markerHeight="7"
    refX="0" refY="3.5" orient="auto">
      <polygon points="0 0, 10 3.5, 0 7" fill="#333" />
    </marker>
</svg>


などにより直列の命令列に
直した形式．

この変換は **lowering** と呼ばれていて $^{[1]}$ 現在は Scheme (FemtoLisp) で実装．
次世代実装: [c42f/JuliaLowering.jl](https://github.com/c42f/JuliaLowering.jl) が進行中 $^{[2]}$


<div class="cite">

[1] : 他の処理系ではこれ以外にも低級な表現に直すこと全般を lowering と読んでいることも多い気がしますが， Jula で lowering というとこの (surface-) AST から `CodeInfo` への変換を指す気がします．
[2]: 近いうちに `JuliaLang/` に動く予定なので，将来この資料のリンクが切れていたらそちらを探しに行ってください

</div>

---

<!-- _header: `CodeInfo` -->

<div class="columns-1-2">

<div>

変換例:
<br>


```julia
function f(init, n)
  s = init
  for i in 1:n
      s += @fastmath sin(i)
  end
  return s
end
```

<div class="center">

→

</div>


✅ マクロや `for` などが解体されて直列の命令列になっている



</div>


<div>

```julia
julia> code_lowered(f, (Float64, Int))
1-element Vector{Core.CodeInfo}:
 CodeInfo(
1 ─ %1  = init
│         s = %1
│   %3  = Main.:(:)
│   %4  =   dynamic (%3)(1, n)
│         @_4 = Base.iterate(%4)
│   %6  = @_4
│   %7  =   builtin %6 === nothing
│   %8  =   dynamic Base.not_int(%7)
└──       goto #4 if not %8
2 ┄ %10 = @_4
│         i = Core.getfield(%10, 1)
│   %12 =   builtin Core.getfield(%10, 2)
│   %13 = Main.:+
│   %14 = s
│   %15 = Main.Base
│   %16 =   dynamic Base.getproperty(%15, :FastMath)
│   %17 =   dynamic Base.getproperty(%16, :sin_fast)
│   %18 = i
│   %19 =   dynamic (%17)(%18)
│         s = (%13)(%14, %19)
│         @_4 = Base.iterate(%4, %12)
│   %22 = @_4
│   %23 =   builtin %22 === nothing
│   %24 =   dynamic Base.not_int(%23)
└──       goto #4 if not %24
3 ─       goto #2
4 ┄ %27 = s
└──       return %27
)
```

</div>

</div>

---


<!-- _header: `CodeInfo` -->

<div class="columns">

<div>


✅ 各種最適化のための情報も含まれている

- 導入される変数
- inline 化するときのコスト
- FFI 呼び出しがあるか
...
- **各変数，返り値の型情報**
  <span style="font-size: 0.8em;"><span class="gray">(lowering された段階ではまだ型推論はされていないので <br>何も入っていない)
</span></span>

</div>


<div>

```julia
julia> ci = code_lowered(f, (Float64, Int)) |> first;

julia> propertynames(ci)
 (:code, :debuginfo, :ssavaluetypes, :ssaflags,
  :slotnames, :slotflags, :slottypes, :rettype,
  :parent, :edges, :min_world, :max_world,
  :method_for_inference_limit_heuristics,
  :nargs, :propagate_inbounds, :has_fcall,
  :has_image_globalref, :nospecializeinfer,
  :isva, :inlining, :constprop, :purity, :inlining_cost)

julia> ci.slotnames
6-element Vector{Symbol}:
 Symbol("#self#")
 :init
 :n
 Symbol("")
 :s
 :i


julia> println(ci.slottypes)
nothing
```

</div>

</div>


---

<!-- _header: `CodeInfo` -->


<div class="columns-1-2">

<div>

このあと `CodeInfo` から型推論・各種最適化が行われる．

<span style="font-size: 0.8em;">※ 型推論自体についてはこの後詳しく扱います．</span>

</div>

<div>

```julia
julia> code_typed(f, (Float64, Int), optimize=false)
1-element Vector{Any}:
 CodeInfo(
1 ─ %1  = init::Float64
│         (s = %1)::Float64
│   %3  = Main.:(:)::Core.Const(Colon())
│   %4  =   dynamic (%3)(1, n)::Core.PartialStruct(UnitRange{Int64}, Any[Core.Const(1), Int64])
│         (@_4 = Base.iterate(%4))::Union{Nothing, Tuple{Int64, Int64}}
│   %6  = @_4::Union{Nothing, Tuple{Int64, Int64}}
│   %7  =   builtin (%6 === nothing)::Bool
│   %8  = intrinsic Base.not_int(%7)::Bool
└──       goto #4 if not %8
2 ┄ %10 = @_4::Tuple{Int64, Int64}
│         (i = Core.getfield(%10, 1))::Int64
│   %12 =   builtin Core.getfield(%10, 2)::Int64
│   %13 = Main.:+::Core.Const(+)
│   %14 = s::Float64
│   %15 = Main.Base::Core.Const(Base)
│   %16 =   dynamic Base.getproperty(%15, :FastMath)::Core.Const(Base.FastMath)
│   %17 =   dynamic Base.getproperty(%16, :sin_fast)::Core.Const(Base.FastMath.sin_fast)
│   %18 = i::Int64
│   %19 =   dynamic (%17)(%18)::Float64
│         (s = (%13)(%14, %19))::Float64
│         (@_4 = Base.iterate(%4, %12))::Union{Nothing, Tuple{Int64, Int64}}
│   %22 = @_4::Union{Nothing, Tuple{Int64, Int64}}
│   %23 =   builtin (%22 === nothing)::Bool
│   %24 = intrinsic Base.not_int(%23)::Bool
└──       goto #4 if not %24
3 ─       goto #2
4 ┄ %27 = s::Float64
└──       return %27
) => Float64
```

<div class="center">

optimize せず単に型推論した結果

</div>

</div>


---

<!-- _header: 型推論による最適化例 -->

<br>

✅ `s`, `sin(i)` が `Float64` なことが示せるので `+` で呼び出す**メソッドを runtime で検索しなくても**
**直接呼びされされるメソッド** (`Base.add_float`) にして高速化できる！ <span style="font-size: 0.8em;"></span>
<span style="font-size: 0.8em;">

<div class="columns">

<div>

```julia
julia> code_typed(f, (Float64, Int), optimize=false)
1-element Vector{Any}:
 CodeInfo(
1 ─ %1  = init::Float64
│         (s = %1)::Float64
│   %3  = Main.:(:)::Core.Const(Colon())
...
│   %13 = Main.:+::Core.Const(+)
│   %14 = s::Float64
│   %15 = Main.Base::Core.Const(Base)
│   %16 =   dynamic Base.getproperty(%15, :FastMath)::Core.Const(Base.FastMath)
│   %17 =   dynamic Base.getproperty(%16, :sin_fast)::Core.Const(Base.FastMath.sin_fast)
│   %18 = i::Int64
│   %19 =   dynamic (%17)(%18)::Float64
│         (s = (%13)(%14, %19))::Float64  # <-- ここが `+` の dynamic dispatch
│         (@_4 = Base.iterate(%4, %12))::Union{Nothing, Tuple{Int64, Int64}}
│   %22 = @_4::Union{Nothing, Tuple{Int64, Int64}}
│   %23 =   builtin (%22 === nothing)::Bool
│   %24 = intrinsic Base.not_int(%23)::Bool
└──       goto #4 if not %24
3 ─       goto #2
4 ┄ %27 = s::Float64
└──       return %27
) => Float64
```

<div class="center">

optimize せず単に型推論した結果

</div>

</div>

<div>

```julia
julia> code_typed(f, (Float64, Int), optimize=true)
1-element Vector{Any}:
 CodeInfo(
1 ── %1  = intrinsic Base.sle_int(1, n)::Bool
└───       goto #3 if not %1
...
│    %18 = φ (#9 => %14, #14 => %29)::Int64
│    %19 = φ (#9 => init, #14 => %22)::Float64
│    %20 = intrinsic Base.sitofp(Float64, %17)::Float64
│    %21 =    invoke Base.Math.sin(%20::Float64)::Float64
│    %22 = intrinsic Base.add_float(%19, %21)::Float64 # <-- `Float64` 用の加算
│    %23 =   builtin (%18 === %5)::Bool
└───       goto #12 if not %23
11 ─       goto #13
12 ─ %26 = intrinsic Base.add_int(%18, 1)::Int64
└───       goto #13
...
│    %30 = φ (#11 => true, #12 => false)::Bool
│    %31 = intrinsic Base.not_int(%30)::Bool
└───       goto #15 if not %31
14 ─       goto #10
15 ┄ %34 = φ (#13 => %22, #9 => init)::Float64
└───       return %34
) => Float64
```

<div class="center">

optimize した結果

</div>

</div>



</span>


---

<!-- _header: Julia の型推論 -->

✅ Julia の型推論は **データフロー解析** として解かれる．

⚠️ ただし 使っている lattice は Runtime で使う型の lattice と一致しているわけではなく，
事実上ごく簡単な定数畳み込みに当たる処理も行う lattice も含めて複数の単純な lattice から構成される

<div class="center">

⇩

</div>


---

<!-- _header: Julia の型推論 -->

<div style="font-size: 0.90em">


使う lattice は

- `JLTypeLattice` ... Runtime で使う型の lattice
- `ConstLattice`　 ... 定数情報の lattice
- `PartialsLattice` ... Struct のフィールドの型情報を扱う lattice
- `ConditionalsLattice` ... 条件分岐の情報を扱う lattice

から <span class="dot-text">構成</span> したもの

⇨ `Int` や `Float64` などだけでなく `Const(1)` のようなものまで推論できる．

<span style="font-size: 0.8em;">(⚠️ 全ての定数関連の最適化がここでされているわけではない)</span>


```julia
# Compiler/src/abstractlattice.jl
const SimpleInferenceLattice = typeof(PartialsLattice(ConstsLattice()))
const BaseInferenceLattice = typeof(ConditionalsLattice(SimpleInferenceLattice.instance))

# Compiler/src/types.jl
typeinf_lattice(::AbstractInterpreter) = InferenceLattice(BaseInferenceLattice.instance)
```

</div>



---

<!-- _header: Julia の型推論 -->

構成の理論的な基礎づけは....

**僕はまだ完全にはわかっていません😢** 


---

<!-- _header: Julia の型推論 -->


<div class="columns">

<div>

$\sqcup$ の実行例:


```julia
julia> CC.tmerge(CC.ConstsLattice(),
           CC.Const(3),
           CC.Const(4))
Int64

julia> CC.tmerge(CC.ConstsLattice(),
           CC.Const(3),
           CC.Const(4.0))
Union{Float64, Int64}

julia> CC.tmerge(CC.ConstsLattice(),
           CC.Const(3),
           Int)
Int64

julia> CC.tmerge(CC.ConstsLattice(),
           CC.Const(3),
           Float64)
Union{Float64, Int64}
```

<span style="font-size: 0.8em;">

$c \leq \top$ の間に Type が入っている感じの lattice

</span>


</div>

<div>

$\sqcup$ の実装例:


```julia
# Compiler/src/typelimits.jl#tmerge
function tmerge(lattice::ConstsLattice, typea, typeb)
    acp = isa(typea, Const) || isa(typea, PartialTypeVar)
    bcp = isa(typeb, Const) || isa(typeb, PartialTypeVar)
    if acp && bcp
        typea === typeb && return typea
    end
    wl = widenlattice(lattice)
    acp && (typea = widenlattice(wl, typea))
    bcp && (typeb = widenlattice(wl, typeb))
    return tmerge(wl, typea, typeb)
end
```

<span style="font-size: 0.75em;">
<span class="gray">
※ 見やすさのために最適化ヒントなどを消しています
</span>
</span>


</div>

</div>


---

<!-- _header: Julia の型推論 -->

正当化のいろいろなことは考えていて...例えば一番簡単な例:

<div class="thm" style="font-size: 0.8em;">

<br>

**[Lattice の階層的な構成例 ①]**

Lattice $(L_1, \leq_1), (L_2, \leq_2)$ について ($L_1$ と $L_2$ は互いに素)

$$
\begin{align*}
L &= L_1 \cup L_2 \\
\leq &= \{(x, y) \mid (x, y \in L_1 \land x \leq_1 y) \lor (x, y \in L_2 \land x \leq_2 y) \lor (x \in L_1 \land y \in L_2)\}
\end{align*}
$$

と定めると $(L, \leq)$ も Lattice で，

$$
\begin{array}{l@{\qquad}l}
\displaystyle
\begin{aligned}
a \sqcap b &=
\begin{cases}
a \sqcap_1 b & (a,b\in L_1)\\
a \sqcap_2 b & (a,b\in L_2)\\
a            & (a\in L_1,\ b\in L_2)\\
b            & (a\in L_2,\ b\in L_1)
\end{cases}
\end{aligned}
&
\displaystyle
\begin{aligned}
a \sqcup b &=
\begin{cases}
a \sqcup_1 b & (a,b\in L_1)\\
a \sqcup_2 b & (a,b\in L_2)\\
b            & (a\in L_1,\ b\in L_2)\\
a            & (a\in L_2,\ b\in L_1)
\end{cases}
\end{aligned}
\end{array}
$$

</div>

(ただこれだと $\text{Const}(1) \sqcap \text{Const}(2) = \text{Int}$ 的なことはできない)
(おそらく $L_i$ と $L_{i+1}$ に何らかの関係があることから攻めないといけなさそう)

---

<!-- _header: Julia の型推論 ⚠️ -->

<span style="font-size: 0.8em;">

これを書いていて気がついた役立ちそうな例:

</span>


<div class="thm" style="font-size: 0.8em;">

<br>

**[Lattice の階層的な構成例 ②]**

Complete Lattice $(L_1, \sqcup_1, \sqcap_1), (L_2, \sqcup_2, \sqcap_2)$ と $\alpha: L_2 \to L_1, \gamma: L_1 \to L_2$ について ($L_1$ と $L_2$ は互いに素)

条件: $(L_1, \alpha, \gamma, L_2)$ が Galois connection をなす

は

$$
\begin{array}{l@{\qquad}l}
\displaystyle
\begin{aligned}
a \sqcup b &=
\begin{cases}
a \sqcup_1 b & (a,b\in L_1)\\
a \sqcup_2 b & (a,b\in L_2)\\
a \sqcup_1 \alpha(b) & (a\in L_1,\ b\in L_2)\\
\alpha(a) \sqcup_1 b & (a\in L_2,\ b\in L_1)
\end{cases}
\end{aligned}
&
\displaystyle
\begin{aligned}
a \sqcap b &=
\begin{cases}
a \sqcap_1 b & (a,b\in L_1)\\
a \sqcap_2 b & (a,b\in L_2)\\
\gamma(a) \sqcap_2 b & (a\in L_1,\ b\in L_2)\\
a \sqcap_2 \gamma(b) & (a\in L_2,\ b\in L_1)
\end{cases}
\end{aligned}
\end{array}
$$

により定めた $(L, \sqcup, \sqcap)$ が Complete Lattice になることと同値．

</div>

<span style="font-size: 0.8em;">

(proof) wip

<!-- (proof) 以前のゼミの内容の一部を Lean で形式化したのでその続きで書いてみました: https://github.com/abap34/galois-connection.lean/blob/main/GaloisConnection/Basic.lean <span style="color:red">(※ lean については本当に初心者なのでそこは参考にしないでください)</span> -->


</span>


---

<!-- _header: Julia の型推論  -->

**需要**:
できれば簡単に lattice を差し替えて必要な解析をやりたい．
(ほとんどの場合 abstract semantics を再利用できるだろうから)

**現状**:
結構大変．今後の改善に期待 👀
(お前がやるんだよ && 研究ネタ案件?)


---

<!-- _header: SSA 形式への変換と最適化⚠️ -->

<span class="gray" style="position: fixed; top: 10px; left: 20px; font-size: 0.6em; text-align: left;">

(ここ以降は自分があまり触ったことのない範囲なので詳しい実装レベルまではわかっていません)

</span>



✅ さらなる最適化のために SSA 形式への変換と最適化が行われている．

- Dead Code の削除
- エスケープ解析
  - ヒープアロケーションの削減
- SROA (Scalar Replacement of Aggregates)
  - struct を分解することで construct / field アクセスを消す
- 不要と示せる場合の type assersion の削除



![bg right 90%](img/image-3.png)


---


<!-- _header: LLVM IR とコード生成⚠️ -->

✅ 最終的に LLVM IR に変換され，LLVM の最適化パスを通してコード生成される．

LLVM のさまざまな最適化パスを利用. (https://github.com/JuliaLang/julia/blob/a5576b4ddb61ce10eefba752a21ef806c4d6fd93/src/pipeline.cpp)

- 演算の簡約
- 演算子強度低減
- loop の vectorization, unrolling
- libcall のむやみな呼び出しの削除

... (非常に多くの最適化が行われている)