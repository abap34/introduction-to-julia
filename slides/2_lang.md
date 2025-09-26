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

1. Julia のキホンと特徴

---



<!-- _header: Julia のメタ情報を 20秒で -->


- 2009年から開発が始まり， 2012年にリリースされた <i class="bi bi-github"></i>
  - 当時 MIT の博士課程の学生だった Jeff Bezanson を中心に Stefan Karpinski, Viral B. Shah, Alan Edelman などによって開発がスタート
  - 今でもボストンが中心地のイメージ
  - 2018年に v1.0 リリース． 現在の最新リリースは v1.11.6
- (広く使われている) 唯一の処理系: [Julialang/julia](https://github.com/JuliaLang/julia)
  - オープンソース (MIT License)
  - Base やコンパイルプロセスの結構な部分は Julia で書かれている． C も結構． LLVM 近辺で C++ もそこそこ．フロントエンドのためにScheme 少し

---

<!-- _header: Julia のキホンと特徴 -->

<span class="orangelined">**このプログラミング言語は速い！というのはとても難しい**</span>
<span style="font-size: 0.8em;">(ほとんどの速度比較に対して問題点を指摘できる自信があります)</span> が...

<br>


<div class="center">

<span style="font-size: 0.8em;">ざっくり</span>



# Julia は速い

<br>

... はおそらくほぼコンセンサスが得られている

<span style="font-size: 0.8em;">
(少なくとも Python/R よりはずっと速い， C/C++/Fortran と比較対象になれるくらい)</span>

</div>

---

<!-- _header: Julia が速いと何が嬉しいのか？ -->

よくいわれる話:

# Two-Language Problem

---

<!-- _header: Two-Language Problem -->


**科学技術計算側からの需要**:

- 高速に実験をするために，プロトタイピングがしやすい言語を使いたい．
- 職業プログラマではないので，学習コストが低い言語を使いたい．
- 大規模な計算のためにパフォーマンスが良いプログラムが書きたい．

<div class="center">


⇩


</div>


**典型的な解決策:**

1. <span class="dot-text">簡単な</span> 言語でプロトタイピングをする
2. パフォーマンス的に重要な部分を C, C++ などで書き直す



---

<!-- _header: Two-Language Problem -->

<div class="box" style="font-size: 0.75em;">
<br>

**典型的な解決策**:

1. <span class="dot-text">簡単な</span> 言語でプロトタイピングをする
2. パフォーマンス的に重要な部分を C, C++ などで書き直す

</div>

をすると...

1. <span class="orangelined">**開発コストが大きくなる**</span>
   1. ただでさえ職業プログラマじゃないのに 2つの言語を覚えないといけない
   2. 同じロジックを再実装する必要がある
2. <span class="orangelined">**コミュニティ・コード資産の分断**</span>

---


<!-- _header: Two-Language Problem -->

Julia の謳い文句:

## ✅ **プロトタイピング向き** でありながら
## ✅ **ハイパフォーマンス** なプログラムが書ける

---

<!-- _header: そのほかの Julia の特徴 -->

- **多重ディスパッチ** (Multiple Dispatch)
  - 全ての引数の型に基づいて関数の実装を選択する
- **JIT コンパイル**
  - LLVM の ORCv2 を利用して関数単位でコンパイル
  - インタラクティブに使いやすい
- **強力なメタプログラミング** (Metaprogramming)
  - Homoiconic
  - 衛生的マクロ
- **World Age** $^{[1]}$
  - `eval` ありで最適化を安全に行う仕組み
- **パッケージマネージャ同梱**
  - 仮想環境の作成や管理・配布まで容易

<div class="cite">

[1] Belyakova, Julia, et al. "World age in julia: Optimizing method dispatch in the presence of eval." Proceedings of the ACM on Programming Languages 4.OOPSLA (2020): 1-26.


</div>

---