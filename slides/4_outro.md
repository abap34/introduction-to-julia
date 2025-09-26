---
marp: true
paginate: true
math: mathjax
theme: honwaka
---




<!-- _header: 今後の展望 (シェア編) -->

<span style="font-size: 0.75em;">

<br>


- 数値計算向け言語としては少なくともしばらくは滅びはしないだろう
  - 今まで Fortran, MATLAB, C++ とかがシェアを取ってきた領域でシェアをとりつつある
  - アカデミアの人から人気がある
    - 毎年やっている survey $^{[1]}$ によれば research に使っている人が一貫して一番多い 😯
  - そういう意味で、 <span class="red">いま 真にライバルなのは Fortran, MATLAB </span>  (そしてこの勝負は分がいい)
- 期待されていた機械学習ではどう？
  - 世間の人のよくある Julia のイメージはたぶん <span class="red"> Alternative to Python </span>
  - が、機械学習プレイヤーとしては <span class="red">**素直に Python 使ったほうが楽**</span> というのが正直なところ
    - エコシステムが強すぎる
  - 個人的には結構厳しいと思う
- ところでとても好きだし面白い言語だと思う
  - 少なくとも python の静的型チェッカくらいの体験は提供できるように頑張っていきたい

</span>



<br>

<div class="cite">

[1] https://julialang.org/assets/2025-julia-user-developer-survey.pdf

</div>


<span class="red">TODO: 公開時に消す</span>


---

<!-- _header: 今後の展望 (言語・コンパイラ編) -->

- インターフェースか型クラスがあった方がいいと思う
- コンパイラインフラの改善
  - いろんな解析ができたらうれしい
  - グローバルな実行をうまく handle できるようになってほしい
    - こういうのをうまく abstraction するのは難しい
    - 部分的に concretize するための理論と実装が必要
- まともな AOT コンパイル
- (JIT-) コンパイルの高速化
  - Copy-and-Patch とか？
  - なんかいい感じの高速化など


