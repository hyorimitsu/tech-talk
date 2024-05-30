React Compiler について
---

2024 年 5 月 15 日 ~ 5 月 16 日（米国時間）に React Conf 2024 が開催されました。  
React 19 の新機能についてなどさまざまな発表がありましたが（[Recap](https://react.dev/blog/2024/05/22/react-conf-2024-recap) があるので見ていない方はぜひ見てみてください）、ここでは[オープンソース化](https://github.com/facebook/react/pull/29061) ([ソースコード](https://github.com/facebook/react/tree/main/compiler)) が発表された [React Compiler](https://react.dev/learn/react-compiler) について紹介したいと思います。


## 概要

React Compiler は React のアプリケーションをビルド時に自動的に最適化するツールです。  

React を触れていた方であれば馴染み深いと思いますが、今までは `useMemo` や `useCallback`, `memo` を用いて不要な再計算、再レンダリングを抑え、パフォーマンスの最適化を開発者自身が行う必要がありました。

React Compiler ではこれらの最適化を自動的に行ってくれるため、よりシンプルに、より安全に開発を行うことができるようになります。

### NOTE

全ての関数をメモ化するわけではなく、Component と Hook のみをメモ化するようになっているそうです。  

```js
// Component でも Hook でもないため、React Compiler ではメモ化されません
function expensivelyProcessAReallyLargeArrayOfObjects() { /* ... */ }

// Component のため、React Compiler でメモ化されます
function TableContainer({ items }) {
  // 関数呼び出しはメモ化されます
  const data = expensivelyProcessAReallyLargeArrayOfObjects(items);
  // ...
}
```
※ [公式ドキュメントに挙げられていたコード](https://react.dev/learn/react-compiler#expensive-calculations-also-get-memoized)が分かりやすかったので、コメントだけ和訳した形でお借りしています。


## 導入方法

導入方法については各公式ドキュメントで紹介されています。

- [React (新規プロジェクトへの導入)](https://react.dev/learn/react-compiler#new-projects)
  - Vite での導入方法はもちろん、Next.js や Remix といったフレームワークでの導入方法も紹介されています
- [React (既存プロジェクトへの導入)](https://react.dev/learn/react-compiler#existing-projects)
- [Next.js](https://nextjs.org/blog/next-15-rc#react-compiler-experimental)

公式ドキュメントに書かれているとおりですが、既存プロジェクトへの導入については、オプションで対象のディレクトリを絞って部分的に有効にするところから始めることが推奨されています。

また、早期導入者への支援、そしてトラブルシュートのための機能として [`"use memo"`](https://react.dev/learn/react-compiler#existing-projects) や [`"use no memo"`](https://react.dev/learn/react-compiler#something-is-not-working-after-compilation) といったディレクティブも用意されていますが、いずれも長期的に使用されることは意図していないもののようなので、これらは開発時の確認程度の目的での利用にとどめておいた方が良いかもしれません。


## 再レンダリングの様子を見てみる

実際にどのように再レンダリングが行われるのか、以下のような簡単なカウンタアプリで見ていきたいと思います。

```ts
import { useState } from 'react'
import './App.css'

type HeaderProps = {
  title: string
}

const Header = ({ title }: HeaderProps) => {
  return <header style={{ 'margin': '10px' }}>{title}</header>
}

type CounterProps = {
  count: number
  onClick: () => void
}

const Counter = ({ count, onClick }: CounterProps) => {
  return <button style={{ 'margin': '10px' }} onClick={onClick}>Count: {count}</button>
}

function App() {
  const [count1, setCount1] = useState(0)
  const [count2, setCount2] = useState(0)

  return (
    <div style={{ 'padding': '10px' }}>
      <Header title='Counter App'/>
      <Counter count={count1} onClick={() => setCount1(count1 + 1)} />
      <Counter count={count2} onClick={() => setCount2(count2 + 1)} />
    </div>
  )
}

export default App
```

まずは React Compiler を利用していない場合のレンダリングの様子です。  
一方の Counter を更新した際に、もう一方の Counter と Header も再レンダリングされてしまっていることが分かります。

<div style="text-align:center">
  <video src="https://github.com/hyorimitsu/tech-talk/assets/52403055/2ca5750f-5448-4ceb-964e-7ba5f298d793" width="90%" controls="true"></video>
</div><br>

次に、React Compiler を利用した場合のレンダリングの様子です。  
更新した Counter だけが再レンダリングされ、もう一方の Counter と Header は再レンダリングされていません。

<div style="text-align:center">
  <video src="https://github.com/hyorimitsu/tech-talk/assets/52403055/d2906d7f-a49b-409b-af57-92a1c739eb37" width="90%" controls="true"></video>
</div><br>

※ [React Developer Tools (v5.0+)](https://react.dev/learn/react-developer-tools) では React Compiler のサポートが組み込まれており、コンパイラによって最適化されたコンポーネントについては `Memo ✨` と表示されるようになっています。


## ビルド結果を見てみる

概要や導入方法、実際の動作の様子が分かったところで、どのようにビルドされるかを見ていきたいと思います。 
分かりやすいように、最もシンプルな形として、以下のようなコードで確認してみます。

```ts
import './App.css'

function App() {
  return (
    <div className='hoge'>
      Hello, World!
    </div>
  )
}

export default App
```

まずは React Compiler を利用していない場合のビルド結果です。  
`jsxDEV` ([reactjs/rfcs](https://github.com/reactjs/rfcs/blob/createlement-rfc/text/0000-create-element-changes.md), [@babel/plugin-transform-react-jsx-development](https://babeljs.io/docs/babel-plugin-transform-react-jsx-development)) で要素を作成してそれを返却しているだけの、(キャッシュ等はしていないという意味で) 非常にシンプルなものです。

```js
function App() {
  return /* @__PURE__ */ jsxDEV("div", { className: "hoge", children: "Hello, World!" }, void 0, false, {
    fileName: "/app/src/App.tsx",
    lineNumber: 5,
    columnNumber: 5
  }, this);
}
```

次に、React Compiler を利用した場合のビルド結果です。  
コメントで記述しているとおり、キャッシュを用いて要素を再利用する形になっています。

```js
function App() {
  // キャッシュ用の配列を作成 (コンパイラが自動的にサイズ 2 の配列が必要だと判断した)
  const $ = _c(2);
  // インデックス 0 の要素はキャッシュが初期化されているかどうかのフラグを保持している
  if ($[0] !== "61398d2ffa0ca667d083b06304e5e16c0d24ae045aff865fb21f195793b57a4e") {
    // 初回レンダリング時の初期化処理
    for (let $i = 0; $i < 2; $i += 1) {
      $[$i] = Symbol.for("react.memo_cache_sentinel");
    }
    $[0] = "61398d2ffa0ca667d083b06304e5e16c0d24ae045aff865fb21f195793b57a4e";
  }
  // インデックス 1 の要素は、DOM ツリーを持つ JSX のメモ化されたバージョンを保持している
  let t0;
  if ($[1] === Symbol.for("react.memo_cache_sentinel")) {
    // インデックス 1 の要素が初期化シンボルと一致する場合は要素を生成してキャッシュ
    t0 = /* @__PURE__ */ jsxDEV("div", { className: "hoge", children: "Hello, World!" }, void 0, false, {
      fileName: "/app/src/App.tsx",
      lineNumber: 13,
      columnNumber: 10
    }, this);
    $[1] = t0;
  } else {
    // 一致しない場合はキャッシュされた要素を再利用
    t0 = $[1];
  }
  return t0;
}
```

ビルド結果の違いは分かりましたが、`_c` 関数の中身はどうなっているのでしょうか？  
実際に見てみましょう。

```js
// https://github.com/facebook/react/blob/main/compiler/packages/react-compiler-runtime/src/index.ts#L21-L37 より引用
const $empty = Symbol.for("react.memo_cache_sentinel");
/**
 * DANGER: this hook is NEVER meant to be called directly!
 **/
export function c(size: number) {
  return React.useState(() => {
    const $ = new Array(size);
    for (let ii = 0; ii < size; ii++) {
      $[ii] = $empty;
    }
    // This symbol is added to tell the react devtools that this array is from
    // useMemoCache.
    // @ts-ignore
    $[$empty] = true;
    return $;
  })[0];
}
```

devtools 用の処理もありますが、`Symbol.for("react.memo_cache_sentinel")` で初期化した指定サイズの配列を `useState` で作っているだけです。  

また、`useState` で作成した配列をのみを返すようになっており、その配列を呼び出し側で直接操作する形になっています。  
これはセッター関数を呼び出すと再レンダリングが行われてしまうためであり、こうすることで、再レンダリングを回避しつつ、キャッシュの内容を更新しています。


## 補足

React Compiler には、誤ったコードなどが原因でコンパイル中に問題が発生した場合、元のトランスパイラにフォールバックする機能が備わっているようです。  
また、React Compiler には [Linter](https://github.com/facebook/react/tree/main/compiler/packages/eslint-plugin-react-compiler) も用意されています。

フォールバックの具体例を含め、React Compiler の動作について分かりやすく解説している動画があったので、こちらを見ていただくとより理解が深まるかもしれません。  
https://www.youtube.com/watch?v=PYHBHK37xlE


## まとめ

個人的に、React Compiler は React アプリケーションの開発体験が変わる大きなアップデートだと思っています。

現時点では実験的な機能ということでまだ本番環境への導入はできませんが、どうやらすでに Instagram の運用で使用されているらしいので、近い将来に安定版としてリリースされるかもしれません。

弊社では Next.js を利用することが多いので、安定版がリリースされたら積極的に使っていきたいと思います。


## 参考文献

- [React Labs: What We've Been Working On – February 2024 - React Blog](https://react.dev/blog/2024/02/15/react-labs-what-we-have-been-working-on-february-2024#react-compiler)
- [React Conf 2024 Recap - React Blog](https://react.dev/blog/2024/05/22/react-conf-2024-recap)
- [React Compiler - React](https://react.dev/learn/react-compiler)
- [Next.js 15 RC - The latest Next.js news](https://nextjs.org/blog/next-15-rc#react-compiler-experimental)
- [Open-source React Compiler - GitHub](https://github.com/facebook/react/pull/29061)
- [facebook/react/compiler - GitHub](https://github.com/facebook/react/tree/main/compiler)
- [reactjs/rfcs - GitHub](https://github.com/reactjs/rfcs/blob/createlement-rfc/text/0000-create-element-changes.md)
- [@babel/plugin-transform-react-jsx-development - Babel](https://babeljs.io/docs/babel-plugin-transform-react-jsx-development)
- [React Compiler With React 18 - Medium](https://jherr2020.medium.com/react-compiler-with-react-18-1e39f60ae71a)
- [React Conf 2024 Day 2 - YouTube](https://www.youtube.com/watch?v=0ckOUBiuxVY&t=9677s)
