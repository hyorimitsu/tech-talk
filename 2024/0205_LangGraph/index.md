LangGraph とは ~ LangChain を用いたマルチエージェントシステムの構築 ~
---

2024 年 1 月 8 日 に LangChain の初めての stable version である `LangChain v0.1.0` がリリースされました。  
それに合わせて、新しいライブラリである `LangGraph` のリリースも発表されたので、今回はこちらについて簡単に紹介します。

# LangGraph とは

`LangGraph` は LLM と LangChain を使用してステートフルな Multi-Actor アプリケーションを構築するためのライブラリです。

LangChain では LCEL (LangChain Expression Language) を用いることで Chain を構築することができました。  
しかし、これで構築できるものはあくまでも DAG (Directed acyclic graph: 有向非巡回グラフ) であり、サイクルを持たせることができません。

この場合、例えば RAG を用いたアプリケーションではうまく機能しない場合があります。  
最初のデータ取得ステップで有用でない情報が返されたとしても、再取得などが行われることなく次のステップへ進んでしまうためです。

この問題を解決するのが `LangGraph` であり、 LCEL を拡張して複数のステップにまたがる Chain (または Actor) を循環的に調整する機能を備えています。

例えば、以下のようなグラフ構造を構築することができます。

<div style="text-align:center">
  <img src="https://github.com/hyorimitsu/tech-talk/blob/main/2024/0205_LangGraph/img/langgraph_multi_agent_diagram.png?raw=true" alt="langgraph_multi_agent_diagram" />
  <a href="https://blog.langchain.dev/langgraph-multi-agent-workflows/" target="_blank">https://blog.langchain.dev/langgraph-multi-agent-workflows/</a> より引用
</div><br>
<div style="text-align:center">
  <img src="https://github.com/hyorimitsu/tech-talk/blob/main/2024/0205_LangGraph/img/langgraph_supervisor_diagram.png?raw=true" alt="langgraph_supervisor_diagram" />
  <a href="https://blog.langchain.dev/langgraph-multi-agent-workflows/" target="_blank">https://blog.langchain.dev/langgraph-multi-agent-workflows/</a> より引用
</div>

# LangGraph の構成

`LangGraph` は大きく以下の 3 つの要素で構成されます。

- StateGraph
- Node
- Edge

## StateGraph

`StateGraph` はグラフの状態を表す要素で、後述する `Node` や `Edge` で利用するための情報を管理することができます。  
例えば、以下のようなコードで定義することができます。  

```python
class AgentState(TypedDict):
  # メッセージの履歴
  messages: Annotated[Sequence[BaseMessage], operator.add]
  # 次のステップ
  next: str

graph = StateGraph(AgentState)
```

この例では `messages` と `next` を持っていますが、アプリケーションに応じて必要な値を持たせることができます。

また、ここで定義した値は `Node` によって更新 (追加または上書き) されます。  
上記の例の場合は `message` への追加と `next` の上書きが行われることを想定しています。

## Node

`Node` は具体的な処理を行う要素で、`name` と `value` を指定して以下のような記述で追加することができます。

```python
graph.add_node(name, value)
```

各パラメータの説明はそれぞれ下記の通りです。

| 名前 | 説明 |
| --- | --- |
| name | `Edge` を追加するときに `Node` を参照するための文字列。 |
| value | 呼び出される関数または LCEL Runnable。上記の `State` オブジェクトと同じ構造のディクショナリを引数として受け取り、同様の構造で更新された状態を返す必要がある。 |

また、サイクルの終了を表す特別な `Node` として `End` が用意されています。

```python
from langgraph.graph import END
```

## Edge

`Node` 間の関係を示す要素で、これにはいくつかの種類が用意されています。

### The Starting Edge

グラフのエントリーポイントとして特定の `Node` を指定することができる `Edge` です。  
グラフに入力があった際、必ず最初に呼び出されるようになります。

```python
graph.set_entry_point(node_name)
```

### Normal Edge

`NodeA` の後に常に `NodeB` が呼ばれるように指定することができる `Edge` です。

```python
graph.add_edge(node_a_name, node_b_name)
```

### Conditional Edge

ルータの役割を持つ関数を指定し、それによって次の `Node` を決定させることができる `Edge` です。

```python
graph.add_conditional_edges(
  # `Edge` の始点となる `Node`
  node_a_name,
  # 始点 `Node` の出力を基に次にどの `Node` を呼び出すかを決定するルーティング関数
  should_continue,
  # ルーティング関数の戻り値 (文字列) と呼び出し先の `Node` のマッピング
  {
    "continue": node_b_name,
    "end": END
  }
)
```

# グラフの実行

グラフを定義したら、実行をする前にコンパイルを行う必要があります。

```python
app = graph.compile()
```

これで Runnable にコンパイルされます。  
この Runnable は、`.invoke`, `.stream`, `.astream_log` など LangChain の Runnable と同様のメソッドを公開しているため、Chain と同じ方法で呼び出すことができます。

```python
app.invoke({"messages": [HumanMessage(content="東京の天気は？")]})
```

# やってみた

Google Colaboratory で実際に試してみました。  
荒いところはありますが、動作や実装の参考程度にはなるかもしれないので、興味がある方は試してみてください。

https://github.com/hyorimitsu/sample-langgraph/blob/main/langgraph.ipynb

# LangGraph vs AutoGen

マルチエージェントフレームワークとしては既に `AutoGen` があります。  
これとの違いについては公式ブログに以下のように記載されています。

```
The biggest difference in mental model between LangGraph and Autogen is in 
construction of the agents.
LangGraph prefers an approach where you explicitly define different agents 
and transition probabilities, preferring to represent it as a graph.
Autogen frames it more as a "conversation".
We believe that this "graph" framing makes it more intuitive and provides 
better developer experience for constructing more complex and opinionated 
workflows where you really want to control the transition probabilities 
between nodes.
It also supports workflows that aren't explicitly captured by "conversations".
```
<div style="text-align: center; margin-bottom: 20px;">
    <a href="https://blog.langchain.dev/langgraph-multi-agent-workflows/" target="_blank">LangGraph: Multi-Agent Workflows</a> より引用
</div><br>

`AutoGen` はエージェント間のインタラクションを「会話」として表現しているのに対し、`LangGraph` はグラフとして表現するアプローチをとっています。  
これにより、`LangGraph` ではワークフローの各ステップを明確に定義し、その間の遷移確率を細かく管理することができるため、より直感的に複雑なマルチエージェントシステムの構築が可能であるとのことです。

# まとめ

生成 AI が話題になってからしばらく経ちますが、まだまだ発展途上の部分も多くあると思います（といっても物凄い速度で進歩していますが）。  
特に、プロダクションレベルのアプリケーションでどう活かすか、どのように構築するかの部分がまだ未熟な段階な印象です。

そんな中、周辺のエコシステムは継続して盛り上がりを見せており、新しいアプローチやライブラリが絶えず登場しています。  
今回紹介した `LangGraph` もそのひとつであり、これらがアプリケーション開発のさらなる発展に繋がることを期待しています。

# 参考文献

- [LangChain v0.1.0 - LangChain Blog](https://blog.langchain.dev/langchain-v0-1-0/)
- [LangGraph | LangChain](https://python.langchain.com/docs/langgraph)
- [LangGraph - LangChain Blog](https://blog.langchain.dev/langgraph/)
- [LangGraph: Multi-Agent Workflows - LangChain Blog](https://blog.langchain.dev/langgraph-multi-agent-workflows/)
- [GitHub - langchain-ai/langgraph](https://github.com/langchain-ai/langgraph)
