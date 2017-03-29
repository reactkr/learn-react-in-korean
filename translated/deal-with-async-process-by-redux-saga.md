> [kuy](https://github.com/kuy)님이 쓰신 [redux-sagaで非同期処理と戦う](http://qiita.com/kuy/items/716affc808ebb3e1e8ac)라는 글의 번역입니다. 본 번역 글은 [원 작자의 허가하](http://qiita.com/kuy/items/716affc808ebb3e1e8ac#comment-55361d9ba905b600cf68)에 작성되어 있습니다.
>
> 또한, 일본어의 특성상 한국어에서는 잘 안쓰는 한자어가 많고 직역할 경우, 가끔 뉘앙스가 애매해지거나 표현이 장황해지는 경우가 있기에, 이러한 부분은 **의도가 변하지 않는 선에서** 의역을 하고 있습니다.
>
> 번역자 : [Junyoung Choi (Rokt33r)](https://github.com/rokt33r)

# redux-saga로 비동기처리와 분투하다.

<!--redux-sagaで非同期処理と戦う-->

## 시작하는 말

<!--## はじめに-->

Redux는 단일 Store, 불변적인 State, Side effect가 없는 Reducer의 3가지 원칙을 내세운 Flux 프레임워크입니다. 하지만 다른 프레임워크와는 달리 제공하는 것은 최소한으로,Fullstack이라고는 말하기 힘들만큼 얇습니다. 그때문에 모든 면에 있어서 일반적이라고 할만한 사용법이 정리되지 않아서, 어떻게 사용해야할지 방황하는 경우도 적지 않습니다. 그 필두로 말할 수 있는게 **비동기처리**입니다. 커뮤니티는 지금도 여러 방법을 모색하고 있고, 주로 쓰여지는건 [redux-thunk](https://github.com/gaearon/redux-thunk)와 [redux-promise](https://github.com/acdlite/redux-promise)정도 일겁니다. Redux로 한정하지 않는다면 [react-side-effect](https://github.com/gaearon/react-side-effect)도 있습니다. 이건 [Twitter의 모바일 버젼](https://mobile.twitter.com)에 사용되고 있습니다. 어떤걸 써도 비동기처리가 가능하게 되지만, 그것들은 어디까지나 도구로써, 설계의 지침까지는 알려주지 않습니다. **문제는 비동기처리를 어디에 쓸 것 인가, 어떻게 쓸 것 인가, 그리고 어디서 불러와야 하는가 입니다.** Redux를 사용하고있으면 다음과 같은 상황이 고민하고 있지는 않으신가요?

<!--ReduxはSingle Store、immutableなState、副作用のないReducerという３つの原則を掲げたFluxフレームワークです。しかし他のフレームワークと違って提供しているものは最小限で、とてもフルスタックとは言えない薄さです。そのためすべてにおいて定番と言える書き方が定まっているわけでもなく、どうしようか迷ってしまうことも少なくありません。その筆頭とも言えるのが **非同期処理** の扱いです。コミュニティでは今でもさまざまな方向に模索が続いていますが、よく使われているものだと[redux-thunk](https://github.com/gaearon/redux-thunk)、[redux-promise](https://github.com/acdlite/redux-promise)あたりでしょうか。Reduxに限定しないのであれば[react-side-effect](https://github.com/gaearon/react-side-effect)というものもあります。こちらは[Twitterのモバイルウェブ版](https://mobile.twitter.com)で使われていますね。どれを使っても非同期処理が可能になりますが、それはあくまで道具であって、設計の指針までは示してくれません。 **問題は非同期処理をどこに書くのか、どのように書くのか、そしてどこから呼び出すべきか、です。** Reduxを使っていると次のような状況に悩んだことはないでしょうか。-->

-   특정 Action을 기다리다 다른 Action을 dispatch한다.
-   통신처리를 완료를 기다리고, 다른 통신처리를 시작한다.
-   초기화시 데이터를 읽어들이고 싶다.
-   빈번히 발생하는 Action을 모아서 dispatch하고 싶다.
-   다른 프레임워크, 라이브러리와 잘 어울리게 하고 싶다.

<!--- 特定のActionを待って、何か別のActionをdispatchする
-   通信処理の完了を待って、別の通信処理を開始する
-   初期化時にデータを読み込みたい
-   頻繁に発生するActionをバッファしてまとめてdispatchしたい
-   他のフレームワーク、ライブラリとうまく連携したい-->

React + Redux의 아름다운 세계에서 움츠러드는 그 코드들을 어떻게 해야할것인가. 어떻게 싸워야 할것인가. 이 글에는 한가지의 해결방법으로써 [redux-saga](https://github.com/yelouafi/redux-saga)를 소개합니다. redux-saga의 개요와 기본적인 개념에대해 가볍게 설명하고, 익숙한 redux-thunk로 만들어진 코드와 비교해보겠습니다. 조금만 초보적인 셋업방법과 실수하기 쉬운 부분을 설명하고, 후반은 실전적인 redux-saga의 사용법을 소개합니다.

<!--React + Reduxのキレイな世界で肩身の狭い思いをするそれらのコードをどうするべきか。どうやって戦っていけばいいのか。本稿では１つの解決方法として[redux-saga](https://github.com/yelouafi/redux-saga)を紹介します。redux-sagaの概要と基本的な考え方についてじっくり説明し、お馴染みのredux-thunkで実装したときとコードを比較してみます。ちょっとだけ入門的なセットアップ方法やハマリポイントについて述べて、後半は実践的なredux-sagaの使い方を紹介します。-->

참고로 공식 리포지토리에서 [한국어로된 README](https://github.com/redux-saga/redux-saga/blob/master/README_ko.md)가 제공되고 있습니다. 일단 써보고싶다! 라는 분들은 먼저 이쪽 링크도 훑어봐주세요.

<!--ちなみに公式リポジトリには[日本語のREADME](https://github.com/yelouafi/redux-saga/blob/master/README_ja.md)も用意しています。とりあえず使ってみたい！という方は先にそちらに目を通してみてください。-->

## redux-saga란

<!--## redux-saga とは-->

**redux-saga는 Redux로 Sideeffect를 다루기 위한 Middleware입니다.** ... 아마 이대로라면 이해하기 어렵겠지요. 이 구문은 라이브러리의 짧은 설명문이지만, 실은 그다지 본질을 표현하고 있지 못합니다. 그런 이유로 제 나름의 이해와 단어로 설명해보겠습니다.

<!--**redux-sagaはReduxで副作用を扱うためのMiddlewareです。** ・・・ちょっとこのままでは理解できませんね。このフレーズ、ライブラリの短い説明文でもあるんですが、実はあまり本質を表現できていません。というわけで自分なりの理解と言葉で説明を試みます。-->

## redux-saga 란 (다시 설명)

<!--## redux-saga とは（仕切り直し）-->

**redux-saga는 "Task"라는 개념을 Redux로 가져오기위한 지원 라이브러리입니다.** 여기서 말하는 Task란 일의 절차와 같은 독립적인 실행 단위로써, 각각 평행적으로 작동합니다. redux-saga는 이 Task의 실행환경을 제공합니다. 더불어 비동기처리를 Task로써 기술하기 위한 준비물입니다. **"Effect"**와 비동기처리를 동기적으로 쓰는 방법을 제공하고 있습니다. Effect란 Task를 기술하기 위한 커맨드(명령, Primitive)와 같은 것으로, 예를들면 다음과 같은 것들이 있습니다.

<!--**redux-sagaは「タスク」という概念をReduxに持ち込むための支援ライブラリです。** ここで言うタスクというのはプロセスのような独立した実行単位で、それぞれが別々に並行して動作します。redux-sagaはこのタスクの実行環境を提供します。それに加えて非同期処理をタスクとして記述するための道具立てである **「作用（Effects）」** と非同期処理を同期的に書き下す手段も提供してくれます。作用というのはタスクを記述するためのコマンド（命令、プリミティブ）のようなもので、例えば次のようなものがあります。-->

-   `select`: State로부터 필요한 데이터를 꺼낸다.
-   `put`: Action을 dispatch한다.
-   `take`: Action을 기다린다. 이벤트의 발생을 기다린다.
-   `call`: Promise의 완료를 기다린다.
-   `fork`: 다른 Task를 시작한다.
-   `join`: 다른 Task의 종료를 기다린다.
-   ...

<!--- `select`: Stateから必要なデータを取り出す
-   `put`: Actionをdispatchする
-   `take`: Actionを待つ、イベントの発生を待つ
-   `call`: Promiseの完了を待つ
-   `fork`: 別のタスクを開始する
-   `join`: 別のタスクの終了を待つ
-   ...-->

이러한 처리중에는 Task안에 직접 실행할 수 있는 것도 있지만, redux-saga에게 부탁하여 간접적으로 실행합니다. 이로써 **비동기처리를 [co](https://github.com/tj/co)처럼 동기적으로 쓸 수 있게 하고, 복수의 Task를 동시평행적으로 실행**하는 것이 가능합니다. 다음 그림은 redux-saga에서 실행되는 Task의 이미지입니다.

<!--これらの処理の中にはタスク内で直接実行できるものもありますが、redux-sagaに依頼することで間接的に実行します。それによって **非同期処理を[co](https://github.com/tj/co)のように同期的に書けるようにしつつ、複数のタスクを同時並行に実行する** ことができます。次の図はredux-saga上で実行されるタスクのイメージです。-->

![redux-saga.png](https://qiita-image-store.s3.amazonaws.com/0/69860/8cc1a873-c675-9009-570d-9684da4a704f.png)

### 무엇이 좋아지나?

<!--### 何がうれしいのか-->

Flux나 Redux만으로도 난해했던것이 더욱 새로운 개념을 가져와서 혼란스럽게 하고있네요. 그래도 Redux-saga를 쓸 가치는 있다고 생각합니다.

<!--FluxやReduxだけでもややこしいのに、さらにいろいろと新しい概念を持ち込まれて混乱してしまいますね。それでもredux-sagaを使う価値はあると思っています。-->

-   Mock 코드를 많이 쓰지 않아도 된다.
-   작은 코드로 더 분할할 수 있다.
-   재이용이 가능해진다.

<!--- モックのコードをたくさん書きたくない
-   小さなコードにどんどん分割できる
-   再利用可能になる-->

이것은 단순한 "재이용가능"이라는 단어 이상의 의미가 있습니다. 무슨 말이냐면 재이용가능한 Container Component를 개발하는데 있어 필수적인 요소입니다. Middleware라는 정말 이해하기 힘들지만 신경쓸 수 밖에 없는 것이 산더미처럼 있고, 더욱이 재이용가능한 컴포넌트로써 도입할떄도, 어디에 Middleware를 넣어줄 것인가를 생각하지않으면 안됩니다. 반면, saga라면 원칙적으로 서로 독립적을 동작하기 때문에 자신의 세계에서만 코드를 쓰는게 가능하여, 다른 saga에 영향을 끼치지 않습니다.

<!--これは単純な「再利用可能」という言葉以上の意味があります。どういうことかというと再利用可能なContainerコンポーネントを開発する上で不可欠な要素だからです。Middlewareというのは本当にやっかいで気にしなければならないことが山ほどあって、さらに再利用可能なコンポーネントとして導入する際にも、どの位置にMiddlewareを組み込むか考えないといけません。一方でSagaであれば原則的にお互いに独立して動作するので自分の関心のある世界だけでコードを書くことができて、他のSagaに影響を与えることがありません。-->

추상적인 설명은 그다지 이해도가 높아지지 않기에, redux-thnk로 쓴 코드와 비교하며 redux-saga로 인해 어떻게 바뀌는지를 한번 봅시다.

<!--抽象的な説明だとなかなか理解が進まないと思うので、redux-thunkで書いたコードと比較しながらredux-sagaによってどのように変わるのか見ていきましょう。-->

## redux-thunk → redux-saga

샘플로써 [FetchAPI](https://r.mozilla.org/en-US/docs/Web/API/Fetch_API)를 사용한 통신처리를 해봅시다.

<!--サンプルとして[Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)を使った通信処理を考えてみます。-->

데이터를 읽어들이는건 간단하지만, Redux로 제대로 생각해야할게 적지 않습니다. 예를 들면 다음과 같은 점들입니다.

<!--データを読み込むのは簡単なんですが、Reduxでちゃんとやろうとすると考えるべきことは少なくありません。例えば次のような点です。-->

-   어디서 통신처리를 적을 것인가
-   어디로부터 통신처리를 불러올 것인가
-   통신처리의 상태를 어떻게 가지게 할 것인가.

<!--- どこに通信処理を書くか
-   どこから通信処理を呼び出すか
-   通信処理の状態をどう保持するか-->

3번째 요소는 redux-thunk나 redux-saga 둘다 공통적인 부분이므로 먼저 설명하겠습니다.

<!--３つ目はどちらのコードでも共通の考え方なので先に説明しておきます。-->

통신처리가 완료되기까지 "읽어들이는 중..."이라는 메세지를 표시하기 위해서는, 통신상태를 Store로 가지게 한 다음에, 통신의 개시/성공/실패의 3 타이밍에 Action을 dispatch하여 상태를 바꿀 필요가 있습니다. 이 적용패턴인 Redux의 [샘플 코드](https://github.com/reactjs/redux/blob/master/examples/real-world/src/actions/index.js#L3-L5)가 아마 원본으로, 이것만 뽑아낸 [redux-api-middleware](https://github.com/agraboso/redux-api-middleware)라는 라이브러리가 있지만, 이번엔 Middleware나 redux-api-middleware를 사용하지 않고 쓰여있습니다. 통신상태뿐만 아니라, 통신이 정상적으로 종료되었는가, 에러로 인해 종료되었는가까지 맞추어 넣어두면 에러메세지를 표시하는데 쓸 수 있어서 편리합니다.

<!--通信処理が完了するまで「読込中...」のようなメッセージを表示するためには、通信の状態をStoreで保持した上で、通信の開始・成功・失敗の３つのタイミングでActionをdispatchして状態を変化させる必要があります。この実装パターンはReduxの[サンプルコード](https://github.com/reactjs/redux/blob/master/examples/real-world/src/actions/index.js#L3-L5)がおそらくオリジナルで、それを切り出した[redux-api-middleware](https://github.com/agraboso/redux-api-middleware)というライブラリもありますが、今回はMiddlewareやredux-api-middlewareを使わずに書いています。通信の状態だけでなく、通信が正常に終了したのか、エラーによって終了したのかも合わせて格納しておくとエラーメッセージの表示に使うことができて便利です。-->

샘플 코드는 3가지의 Action 타입 `REQUEST_USER`, `SUCCESS_USER`, `FAILURE_USER`의 문자열 상수와 Action 오브젝트를 생성하기위한 3가지의 Action Creator `requestUser`,  `successUser`, `failureUser`는 `actions.js`는 `actions.js`에 정의되어 있습니다.

<!--サンプルコードでは３つのAction Type `REQUEST_USER`、`SUCCESS_USER`、`FAILURE_USER` の文字列定数と、Actionオブジェクトを生成するための３つのAction Creator `requestUser`、`successUser`、`failureUser` は `actions.js` で定義済みとします。-->

그럼 redux-thunk 코드를 봅시다.

<!--ではredux-thunkのコードを見てみましょう。-->

### redux-thunk

`api.js`

```js
export function user(id) {
  return fetch(`http://localhost:3000/users/${id}`)
    .then(res => res.json())
    .then(payload => { payload })
    .catch(error => { error });
}
```

`actions.js`

```js
export function fetchUser(id) {
  return dispatch => {
    dispatch(requestUser(id));
    API.user(id).then(res => {
      const { payload, error } = res;
      if (payload && !error) {
        dispatch(successUser(payload));
      } else {
        dispatch(failureUser(error));
      }
    });
  };
}
```

먼저 처리 전체의 흐름을 확인해봅시다.

<!--まずは処理全体の流れを確認してみます。-->

1.  _(누군가가 `fetchUser` 함수의 리턴 값을 dispatch한다)_
2.  redux-thunk의 Middleware가 dispatch된 함수를 실행한다.
3.  통신처리를 개시하기 전에 `REQUEST_USER` Action을 dispatch한다.
4.  `API.user` 함수를 불러내서 통신처리를 개시한다.
5.  완료되면 `SUCCESS_USER` 혹은 `FAILURE_USER` Action을 dispatch한다.

<!--1. *(誰かが `fetchUser` 関数の戻り値をdispatchする)*
2.  redux-thunkのMiddlewareがdispatchされた関数を実行する
3.  通信処理を開始する前に `REQUEST_USER` Actionをdispatchする
4.  `API.user` 関数を呼び出して通信処理を開始する
5.  完了したら `SUCCESS_USER` または `FAILURE_USER` Actionをdispatchする-->

`api.js`의 `user`함수는 유저정보를 가져오는 함수이다. Fetch API는 Promise를 돌려주기에 적절히 처리할 필요가 있습니다. 에러핸들링의 방법은 원하는 대로 하셔도 되지만, 여기선 `try/catch`를 쓰지 않고, 리턴 값으로 판정하는 스타일을 사용하겠습니다.

<!--`api.js` の `user` 関数はユーザー情報を取得する関数です。Fetch APIはPromiseを返すので適切に処理してやる必要があります。エラーハンドリングの方法は好みで構いませんが、今回は `try/catch` を使用せずに戻り値で判定するスタイルを採用しています。-->

`actions.js`의 `fetchUser`함수는 Action Creator이지만, redux-thunk로 부터 실행되기 위해선, Action 오브젝트가 아니라 함수를 돌려줍니다. redux-thunk는 dispatch뿐만 아니라 getState도 파라미터로 넘겨주지만, 지금은 필요하지 않으므로 생략합니다. 좀 전의 통신처리의 패턴에 따라 처음에는 `REQUEST_USER` Action을 dispatch하고, 완료하거나 실패하면 `SUCCESS_USER` 혹은 `FAILURE_USER` Action을 dispatch합니다. 이렇게 redux-thunk를 사용하면 비동기처리 코드를 Action Creator에다 적게 됩니다. 본래의 Action Creator는 Action 오브젝트를 생성하여 돌려줄 뿐이었기에,생성한 Action 오브젝트를 dispatch하는 데다가, 앞뒤로 이런저런 로직이 들어가는건 위험한 냄새가 납니다. 도입도 사용법도 간단한 반면, 사태파악도 안된채로 편하니까 막쓰다간 나중에 지옥을 맞이할지도 모릅니다. 소극적으로 사용한다면 문제가 없겠지만, 복잡한 통신처리는 정말로 쓰고 싶지 않습니다. 쓰고 싶다는 생각도 안됩니다.

<!--`actions.js` の `fetchUser` 関数はAction Creatorですが、redux-thunkに実行してもらうためにActionオブジェクトを返さずに関数を返します。redux-thunkはdispatchだけでなくgetStateもパラメータとして渡してくれますが、今回は不要なので省略しています。さきほどの通信処理の実装パターンに従って最初に `REQUEST_USER` Actionをdispatchして、完了または失敗したら `SUCCESS_USER` または `FAILURE_USER` Actionをdispatchします。このようにredux-thunkを使うと非同期処理のコードをAction Creatorに書くことになります。本来のAction CreatorはActionオブジェクトを生成して返すだけなので、生成したActionオブジェクトをdispatchして、さらに処理の前後にいろいろとロジックが書けてしまうのは危険な香りがしますね。導入も使い方も簡単なのでお手軽な反面、右も左も分からない状態で便利だからといって使いまくると後で地獄を見るかもしれません。控えめに使うのであれば問題ありませんが、とてもこれで複雑な通信処理を書きたいとは思えません。思ってはいけません。-->

그럼 redux-saga로 바꿔쓰면 어떻게 되는지 봅시다.

<!--それではredux-sagaで書き換えるとどうなるか見てみます。-->

### redux-saga

`sagas.js`

```js
function* handleRequestUser() {
  while (true) {
    const action = yield take(REQUEST_USER);
    const { payload, error } = yield call(API.user, action.payload);
    if (payload && !error) {
      yield put(successUser(payload));
    } else {
      yield put(failureUser(error));
    }
  }
}

export default function* rootSaga() {
  yield fork(handleRequestUser);
}
```

밀도가 높은 코드이므로 기합을 넣고 살펴보겠습니다. 먼저 전체적인 흐름부터..

<!--密度の高いコードになっているので気合を入れて見ていきます。まずは全体の流れから。-->

1.  redux-saga의 Middleware가 `rootSaga` Task를 시작시킨다.
2.  `fork` Effect로 인해 `handleRequestUser` Task가 시작된다.
3.  `take` Effect로 `REQUEST_USER` Action이 dispatch되길 기다린다.
4.  _(누군가가 `REQUEST_USER` Action을 dispatch한다.)_
5.  `call` Effect로 `API.user` 함수를 불러와서 통신처리의 완료를 기다린다.
6.  _(통신처리가 완료된다.)_
7.  `put` Effect를 사용하여 `SUCCESS_USER` 혹은 `FAILURE_USER` Action을 dispatch한다.
8.  while 루프에 따라 3번으로 돌아간다.

<!--1. redux-sagaのMiddlewareが `rootSaga` タスクを起動する
2.  `fork` 作用によって `handleRequestUser` タスクが起動する
3. `take` 作用で `REQUEST_USER` Actionがdispatchされるのを待つ
4. _(誰かが `REQUEST_USER` Actionをdispatchする)_
5. `call` 作用で `API.user` 関数を呼び出して、通信処理の完了を待つ
6. _(通信処理が完了する)_
7. `put` 作用を使って `SUCCESS_USER` または `FAILURE_USER` Actionをdispatchする
8. whileループによって3番に戻る-->

redux-thunk로 인한 코드와의 비교를 위해 일련의 흐름을 써보았지만, 실은 이 처리로는 동시병행하여 움직이는 2개의 흐름이 있습니다. 그것이 Task입니다. `sagas.js`로 정의된 2가지 함수 모두 redux-saga의 Task입니다. 하나씩 살펴봅시다.

<!--redux-thunkによるコードとの比較のため一連の流れのように書きましたが、実はこの処理には同時並行に走る２つの流れがあります。それがタスクです。`sagas.js` に定義されている２つの関数はどちらもredux-sagaのタスクです。１つずつ見ていきます。-->

`rootSaga` Task는 Redux의 Store가 작성 된 후, redux-saga의 Middleware가 기동 될 때 1번만 불러와집니다. 그리고 `fork` Effect을 사용하여 redux-saga에게 다른 Task를 기동할 것을 요청합니다. 앞서 설명한듯이, Task내에는 실제 처리를 행하지 않으므로, `fork`함수로부터 생성된 것은 단순한 오브젝트입니다. 이것은 Flux 아키텍쳐의 Action 오브젝트와 가까운 느낌입니다. 그러므로 다음과같이 오브젝트의 내용을 보는 것도 가능합니다.

<!--`rootSaga` タスクはReduxのStoreが作成されたあと、redux-sagaのMiddlewareが起動するときに１回だけ呼び出されます。そして `fork` 作用を使ってredux-sagaに別タスクの起動を依頼します。前述の通り、タスク内では実際の処理は行わないため、`fork` 関数を呼び出して生成されるのはただのオブジェクトです。これはFluxアーキテクチャのActionオブジェクトに近い感じです。そのため次のようにしてオブジェクトの中身を見ることもできます。-->

```js
console.log(fork(handleRequestUser));
```

이를 실행하면, 다음과 같은 느낌의 오브젝트가 생성됩니다.

<!--これを実行すると、次のような感じのオブジェクトが生成されます。-->

```js
{
  Symbol<IO>: true,
  FORK: {
    context: ...,
    fn: <func>,
    args: [...]
  }
}
```

일단, Effect 오브젝트는 생성될뿐 아무것도 하지않으므로, redux-saga로 넘겨져서 실행될 필요가 있습니다. 이 실행을 위해서는 Generator 함수의 `yield`를 사용해 불러오는 코드 값을 넘겨주고 있습니다. "불러오는 쪽의 코드"는 누구의 것인가? 그것은 redux-saga가 제공하는 Task의 실행환경인 Middleware입니다. Effect 오브젝트를 받아들인 redux-saga의 Middleware는 넘겨진 함수를 새로운 Task로써 기동시킵니다. 그 이후 redux-saga는 2개의 Task가 동시에 움직이고있는 상태가 됩니다. 새롭게 기동된 `handleRequestUser` Task의 설명으로 넘어가기 전에 `rootSaga` Task의 "그 이후"를 따라갑니다.

<!--さて、作用オブジェクトは生成しただけでは何も起きないため、redux-sagaに渡して実行してもらう必要があります。この実現のためにGenerator関数の `yield` を使って呼び出し側のコードに値を渡しています。「呼び出し側のコード」というのは誰のことでしょうか？ それはredux-sagaが提供するタスク実行環境であるMiddlewareです。作用オブジェクトを受け取ったredux-sagaのMiddlewareは、渡された関数を新しいタスクとして起動します。これ以降、redux-sagaは２つのタスクが同時に動いている状態になります。新しく起動した `handleRequestUser` タスクに話を移す前にもうちょっと `rootSaga` タスクの「その後」を追います。-->

`fork` Effect는 지정한 Task의 완료를 기다리지 않습니다. 그러므로 `yield`는 블록되지않고 곧 제어로 돌아옵니다. 하지만 `rootSaga` Task는 `handleRequestUser` Task의 기동 이외에 할 일이 없습니다. 그때문에 `rootSaga` Task내에는 `fork`를 사용하여 기동된 모든 Task가 종료 될 때까지 기다립니다. 이러한 움직임은 redux-saga v0.10.0부터 도입된 새로운 실행모델로 인한 것으로, 연쇄적인 Task의 취소를 구현하기위해 필요했습니다. 이것은 부모 Task, 자식 Task, 손자 Task 3개에서, 부모가 자식을 Fork하고, 자식이 손자를 Fork 할 때 부모 Task를 취소하면 제대로 손자Task까지 취소를 알려주는 편리한 기능입니다. 만약 자식Task의 완료를 의도적으로 기다리고 싶지 않다면 `spawn`을 사용하여 Task를 기동해주세요.

<!--`fork` 作用は指定したタスクの完了を待ちません。そのため `yield` するとブロックせずにすぐに制御が戻ってきます。しかし `rootSaga` タスクには `handleRequestUser` タスクの起動以外にやるべきことがありません。そのため `rootSaga` タスク内で `fork` を使って起動したすべてのタスクが終了するまで待ちます。この挙動はredux-saga v0.10.0から導入された新しい実行モデルによるもので、連鎖的なタスクのキャンセルを実現するために必要でした。これは親タスク、子タスク、孫タスクがの３つがあって、親が子をフォークして、子が孫をフォークしたときに親タスクをキャンセルするとちゃんと孫タスクまでキャンセルが伝搬してくれる便利機能です。もし子タスクの完了を意図的に待ちたくないのであれば `spawn` 作用を使ってタスクを起動してください。-->

`handleRequestUser` Task를 기동시키면 금방 `REQUEST_USER` Action을 기다리기 위해 `take` Effect가 불러와집니다. 이 "기다림"이라는 행동이 **비동기처리를 동기적으로 쓴다** 라는 특징적인 Task의 표현으로 이어집니다. redux-saga의 Task를 Generator함수로 쓰는 이유는 `yield` 에 따라 처리의 흐름을 일시정지하기 때문입니다. 이러한 체계 덕분에 싱글 스레드의 Javascript로 복수의 Task를 만들어, 각각 특정한 Action을 기다리거나, 통신처리의 결과를 기다려도 처리가 밀리지 않게 됩니다.

<!--`handleRequestUser` タスクが起動されるとすぐに `REQUEST_USER` Actionを待つために `take` 作用を呼び出します。この「待つ」という挙動が **非同期処理を同期的に書く** という特徴的なタスクの記述につながります。redux-sagaのタスクをGenerator関数で書く理由は `yield` によって処理の流れを一時停止するためです。この仕組みのおかげでシングルスレッドのJavaScriptで複数のタスクを立ち上げて、それぞれで特定のActionを待ったり、通信処理の結果待ちをしても処理が滞ることはありません。-->

`REQUEST_USER` Action이 dispatch되면 `take` Effect를 `yield` 하여 일시정지된 코드가 재개되고, dispatch된 Action 오브젝트를 돌려줍니다. 그리고 곧 API 불러냅니다. 여기서 `call` Effect를 사용합니다. 이것도 다른 Effect와 같이 그 장소에서 실행되지 않는건 공통적이지만, 지정된 함수가 Promise를 돌려줄 경우, 그 Promise 가 resolve되고 나서 제어를 돌려줍니다. `take` Effect와 닮은 움직임이네요. 통신처리가 완려하면 다시 한번 `handleRequestUser` Task로 제어를 돌려주고, 결과에 따라 Action을 dispatch합니다. Action의 dispatch에는 `put` Effect를 사용합니다.

<!--`REQUEST_USER` Actionがdispatchされると `take` 作用を `yield` して一時停止していたコードが再開し、dispatchされたActionオブジェクトが戻り値として返ってきます。そしてようやくAPI呼び出しです。ここで `call` 作用を使います。これも他の作用と同様にその場で実行しないのは共通していますが、指定した関数がPromiseを返す場合、それがresolveしてから制御を返します。`take` 作用と似たような挙動ですね。通信処理が完了すると再び `handleRequestUser` タスクに制御が戻り、結果に応じてActionをdispatchします。Actionのdispatchには `put` 作用を使います。-->

이것으로 통신처리 자체는 완료되었지만, 한가지 더 Task를 정의할 때 자주 쓰는 용어에 대해 설명해두겠습니다. 최초로 코드를 봤을때 "어?" 라고 느끼셨을 거라 생각하지만, `handleRequestUser` Task는 전체가 while문으로된 무한 루프로 감싸여 있습니다. 그 결과 `put` Effect로 Effect로 Action을 dispatch한 후, 루프의 처음으로 돌아가서 다시 한번 `take` Effect로 `REQUEST_USER` Action을 기다리게 됩니다. 즉, Action을 기다려 통신처리를 할 뿐인 Task가 됩니다. 여기가 매우 중요합니다. 이렇게 극단적으로 해야 할 일을 제한해두는 것으로 코드는 매우 단순해집니다. 당연히 버그도 줄겠죠. 게다가 비동기처리에 항상 따라오는 콜백 지옥, 깊은 구조, 뜬금없이 나타나는 Promise가 사라지게 됩니다.

<!--これで通信処理自体は完了なのですが、もう１つだけタスクを定義するときによく使うイディオムについて説明しておきます。最初にコードを見たときに「おや？」と気付いたと思うのですが、`handleRequestUser` タスクは全体がwhile文による無限ループで囲まれています。その結果 `put` 作用でActionをdispatchしたあと、ループの先頭に戻って再び `take` 作用で `REQUEST_USER` Actionを待つことになります。つまりひたすらActionを待って通信処理をするだけのタスクになります。ここ、すごく大事なところです。これくらい極端にやるべきことを絞ってあげるとコードはとても単純でコンパクトになります。当然バグも減りますね。さらに非同期処理に常につきまとうコールバック地獄、深いネスト、突如として出現するPromiseが消えてくれます。-->

### 어떻게 바뀌었나?

<!--### どう変わったか-->

redux-thunk와 redux-saga의 각각의 코드에 대해서 세밀하게 살펴 보았습니다. 여기서 잠깐 다른 관점으로부터 생각해보고 싶습니다. 이 섹션의 머릿글에 말한 "어디에 쓸 것 인가", "어디에서 불러올 것인가"에 대한 얘기입니다.

<!--redux-thunkとredux-sagaのそれぞれのコードについて細かく見ました。ここでちょっと別の観点から考えてみたいと思います。このセクションの冒頭で挙げた「どこに書くか」「どこから呼び出すか」についてです。-->

redux-thunk는 Action Creator가 함수를 넘겨주기에 필연적으로 Action Creator에 비동기처리의 코드나 관련된 로직이 (약간 무리하게) 들어갑니다. 반면, redux-saga는 비동기 처리를 기술하는 전용의 방식인 Task로 쓰여있습니다. 그 결과, Action Creator는 본래의 모습을 되찾아, Action 오브젝트를 생성하여 돌려주는 순수한 상태로 돌아갑니다. 개인적으로 이 변화는 작지 않다고 생각합니다. 그렇게, redux-thunk는 dispatch된 함수를 붙잡아 실행하는 성질상, Middleware 스택(양파같은 구조니까 셸?)의 가장 바깥쪽에 배치될 필요가 있습니다. 그렇지 않으면, 다른 Middleware함수에 붙잡혀 에러가 날지도 모르기 때문입니다. 이러한 사정으로인해 함수가 dispatch되었는지에 대해 redux-thunk이외의 누구도 모르는 사태에 빠지게 됩니다. 만약 redux-thunk의 직전에 redux-logger라든가 다른 Middleware가 오게 되면 이들이 받는건 함수가 됩니다. 내용이 어떻게 될건지 실행하기 전까진 알 수 없습니다. 정말 좋지않네요. redux-saga를 쓴다면 Action Creator가 표준적인 Action 오브젝트를 생성할 뿐이기에 redux-logger로 표시시킬 수 있습니다.

<!--redux-thunkはAction Creatorが関数を投げるので必然的にAction Creatorに非同期処理のコードや関連するロジックを詰め込むことになります。一方でredux-sagaは非同期処理を記述する専用の仕組みであるタスクに書きます。その結果、Action Creatorは本来の姿を取り戻して、Actionオブジェクトを生成して返すだけの素直なヤツに戻ります。個人的にこの変化は小さくないと考えています。というのも、redux-thunkはdispatchされた関数をつかまえてひたすら実行するという性質上、Middlewareスタック（タマネギみたいな構造だからシェル？）の一番外側に配置する必要があります。そうでないと他のMiddlewareが関数をつかまされてエラーを吐くかもしれないからです。このような事情から関数がdispatchされたということはredux-thunk以外誰も知らないという事態に陥ります。もしredux-thunkの手前にredux-loggerか何かを置いたとしても得られるものはただの関数です。中身がどうなっているかは実行してみるまでわかりません。最悪ですね。redux-sagaの方はAction Creatorが標準的なActionオブジェクトを生成するだけなのでredux-loggerで表示できます。-->

그럼, 앞서 통신처리는 매우 단순한 것이었으므로, 처음보는 방식으로 쓰기가 강제된게 버거워 그다지 감사하다라는 인상이 옅었을 지도 모릅니다. 그런 이유로, 현실의 프로젝트에서도 필요할 법한 기능추가를 해봅시다. 복잡할수록 진가를 발휘하는게 redux-saga입니다.

<!--さて、さきほどの通信処理はごく単純なものだったので、見慣れない書き方を強制される方が面倒であまり恩恵を受けた、という印象が薄かったかもしれません。というわけで現実のプロジェクトでも起こりそうな機能追加をやってみましょう。複雑になると真価を発揮するのがredux-sagaです。-->

### 처리를 복잡하게 해보자

<!--### 処理を複雑にしてみる-->

그다지 어려운 처리라면 이해하기 어려워지기 때문에, 이전 [Redux의 middleware를 적극적으로 써보기](http://qiita.com/kuy/items/57c6007f3b8a9b267a8e)라는 포스팅에 응용 예로 만든 [API 요청을 체인시키기](http://qiita.com/kuy/items/57c6007f3b8a9b267a8e#api%E5%91%BC%E3%81%B3%E5%87%BA%E3%81%97%E3%82%92%E3%83%81%E3%82%A7%E3%82%A4%E3%83%B3%E3%81%95%E3%81%9B%E3%82%8B)를 redux-thunk와 redux-saga로 각각 써보겠습니다. 포스팅의 예제은 어떤 통신처리가 끝난 이후, 그 결과에 따라 다시 통신처리를 개시하는 것입니다. 이번 예제에서는 유저 정보를 취득한 이후, 유저정보에 포함된 지역명을 사용하여 같은 지역에 살고 있는 다른 유저를 검색하여 제안하는 기능을 추가해봅니다. 새로운 `api.js`에 추가된 `searchByLocation` 함수는 redux-thunk와 redux-saga로 만든 예제 모두 사용합니다. Action Type이나 Action Creator등은 적당히 정의해뒀다고 생각해주세요.

> 역자주: 예제의 원본이 되는 링크는 일본어이지만 이미 필요한 내용들은 본 포스팅에 다 있기에 모르셔도 크게 문제 없습니다.

<!--あまり難しい処理だと理解しにくくなりますし、以前[Reduxのmiddlewareを積極的に使っていく](http://qiita.com/kuy/items/57c6007f3b8a9b267a8e)という記事で応用例として挙げた[API呼び出しをチェインさせる](http://qiita.com/kuy/items/57c6007f3b8a9b267a8e#api%E5%91%BC%E3%81%B3%E5%87%BA%E3%81%97%E3%82%92%E3%83%81%E3%82%A7%E3%82%A4%E3%83%B3%E3%81%95%E3%81%9B%E3%82%8B)をredux-thunkとredux-sagaのそれぞれで書いてみます。記事のサンプルは、ある通信処理が終わったあと、その結果を元にしてさらに通信処理を開始するというものです。今回のサンプルではユーザー情報を取得したあと、ユーザー情報に含まれる地域名を使って同じ地域に住んでいる他のユーザーを検索してサジェストする機能を追加してみます。新しく `api.js` に追加した `searchByLocation` 関数はどちらのコードでも使用します。Action TypeとかAction Creatorとかは適当に定義してあると思ってください。-->

#### redux-thunk

`api.js`

```js
export function searchByLocation(name) {
  return fetch(`http://localhost:3000/users/${id}/following`)
    .then(res => res.json())
    .then(payload => { payload })
    .catch(error => { error });
}
```

`actions.js`

```js
export function fetchUser(id) {
  return dispatch => {
    // 유저 정보를 읽어들인다
    dispatch(requestUser(id));
    API.user(id).then(res => {
      const { payload, error } = res;
      if (payload && !error) {
        dispatch(successUser(payload));

        // 체인: 지역명으로 유저를 검색
        dispatch(requestSearchByLocation(id));
        API.searchByLocation(id).then(res => {
          const { payload, error } = res;
          if (payload && !error) {
            dispatch(successSearchByLocation(payload));
          } else {
            dispatch(failureSearchByLocation(error));
          }
        });
      } else {
        dispatch(failureUser(error));
      }
    });
  };
}
```

<!--```actions.js
export function fetchUser(id) {
  return dispatch => {
    // ユーザー情報の読み込み
    dispatch(requestUser(id));
    API.user(id).then(res => {
      const { payload, error } = res;
      if (payload && !error) {
        dispatch(successUser(payload));

        // チェイン: 地域名でユーザーを検索
        dispatch(requestSearchByLocation(id));
        API.searchByLocation(id).then(res => {
          const { payload, error } = res;
          if (payload && !error) {
            dispatch(successSearchByLocation(payload));
          } else {
            dispatch(failureSearchByLocation(error));
          }
        });
      } else {
        dispatch(failureUser(error));
      }
    });
  };
}
```-->

...흠. 뭘 하고픈지는 압니다. 아마 보통 이렇게 되겠죠. 하지만 `...`한(뭔가 찝찝한) 느낌이 드네요. 그리고 여기에 또 다른 체인을 늘리거나 체인시키는 위치를 바꾸게 된다면 곤란해 질듯 하네요. 무엇보다도 기분 나쁜건 `fetchUser`라는 Action Creator를 불러내서 왜 유저 검색까지 실행하고 있느냐 라는 점입니다. Middleware를 사용하여 처리를 분리하면 이 문제점이 조금은 해소될 듯 하지만, 어플리케이션 독립적인 DSL과 같은 코드가 계속 늘어나게 되는 것 역시 괴로울 듯 합니다.

<!--・・・うーむ。やりたいことはわかる。それにたぶん普通に書くとそうなる。だけど・・・みたいな気持ちになりますね。そしてここからさらにチェインさせることになったり、チェインさせるタイミングをやっぱり別のとこにする、という要望に答えていくのもつらそうですね。何よりも気持ち悪いのは `fetchUser` というAction Creatorを呼び出してなぜかユーザー検索まで実行されてしまうという点です。Middlewareを使って処理を分離していけばこれらの問題点を多少なりとも解消できそうですが、アプリケーション独自のDSLのようなコードが増えまくってそれはそれでつらそうです。-->

역시, 이 포스팅은 redux-saga를 편애하고 있습니다. 하지만 redux-thunk를 철저하게 까내릴 의도는 없습니다. 실제 저도 아직 사용하는 부분이 있습니다. 어쩔 수 없이 redux-thunk를 계속 사용할 수 밖에 없으신 분도 있을거라 생각되므로, 혹시 "redux-thunk라도 이렇게 쓰면 괴로움을 줄일수 있어!" 같은 의견이 있으시면 코멘트로 알려주세요.

<!--尚、この記事はredux-sagaをえこひいきしています。しかしredux-thunkを徹底的にこき下ろしてやろうという意図はありません。実際私もいまだに使っている部分はあります。やむを得ずredux-thunkを使い続けないといけない方もいると思うので、もし「redux-thunkでもこんな風に書くとつらさが軽減できるよ！」とかありましたらぜひコメントでお知らせ下さい。-->

그럼 redux-saga로 써봅시다.

<!--ではredux-sagaで書いてみましょう。-->

#### redux-saga

`sagas.js`

```js
// 추가
function* handleRequestSearchByLocation() {
  while (true) {
    const action = yield take(SUCCESS_USER);
    const { payload, error } = yield call(API.searchByLocation, action.payload.location);
    if (payload && !error) {
      yield put(successSearchByLocation(payload));
    } else {
      yield put(failureSearchByLocation(error));
    }
  }
}

// 변경없음!
function* handleRequestUser() {
  while (true) {
    const action = yield take(REQUEST_USER);
    const { payload, error } = yield call(API.user, action.payload);
    if (payload && !error) {
      yield put(successUser(payload));
    } else {
      yield put(failureUser(error));
    }
  }
}

export default function* rootSaga() {
  yield fork(handleRequestUser);
  yield fork(handleRequestSearchByLocation); // 추가
}
```

<!--```sagas.js
// 追加
function* handleRequestSearchByLocation() {
  while (true) {
    const action = yield take(SUCCESS_USER);
    const { payload, error } = yield call(API.searchByLocation, action.payload.location);
    if (payload && !error) {
      yield put(successSearchByLocation(payload));
    } else {
      yield put(failureSearchByLocation(error));
    }
  }
}

// 変更なし！
function* handleRequestUser() {
  while (true) {
    const action = yield take(REQUEST_USER);
    const { payload, error } = yield call(API.user, action.payload);
    if (payload && !error) {
      yield put(successUser(payload));
    } else {
      yield put(failureUser(error));
    }
  }
}

export default function* rootSaga() {
  yield fork(handleRequestUser);
  yield fork(handleRequestSearchByLocation); // 追加
}
```-->

보시다싶이 `handleRequestUser` Task는 변경점이 없습니다. 새롭게 추가된 `handleRequestSearchByLocation` Task는 `handleRequestUser` Task와 거의 동일한 처리입니다. `rootSaga` Task는 `handleRequestSearchByLocation` Task를 기동하기위해 `fork` 를 하나 더 추가하고 있습니다. 조금 길어지지만, 처리의 흐름을 다음과 같습니다.

<!--ご覧のように `handleRequestUser` タスクは変更点がありません。新しく追加された `handleRequestSearchByLocation` タスクは `handleRequestUser` タスクとほとんど同じような処理です。`rootSaga` タスクには `handleRequestSearchByLocation` タスクを起動するために `fork` 作用をもう１つ追加しています。ちょっと長いですが、処理の流れを書いておきます。-->

1.  redux-saga의 Middleware가 `rootSaga` Task를 기동시킨다.
2.  `fork` Effect에 따라 `handleRequestUser` 와 `handleRequestSearchByLocation` Task가 기동된다.
3.  각각의 Task에 대해 `take` Effect로부터 `REQUEST_USER` 와 `SUCCESS_USER` Action이 dispatch되는 것을 기다린다.
4.  _(누군가가 `REQUEST_USER` Action을 dispatch한다.)_
5.  `call` Effect으로 `API.user` 함수를 불러와서, 통신처리가 끝나길 기다린다.
6.  _(통신처리가 완료된다.)_
7.  `put` Effect를 사용하여 `SUCCESS_USER` Action을 dispatch한다.
8.  `handleRequestSearchByLocation` 태스크가 다시 시작되어, `call` Effect로 `API.searchByLocation` 함수를 불러와서, 통신처리가 끝나길 기다린다.
9.  _(통신처리가 완료된다.)_
10. `put` Effect를 사용하여 `SUCCESS_SEARCH_BY_LOCATION` Action을 dispatch한다.
11. 각각의 Task에서 while 루프가 처음으로 돌아가 `take` 로 Action의 dispatch를 기다린다.

<!--1.  redux-sagaのMiddlewareが `rootSaga` タスクを起動する
2.  `fork` 作用によって `handleRequestUser` と `handleRequestSearchByLocation` タスクが起動する
3.  それぞれのタスクにて `take` 作用で `REQUEST_USER` と `SUCCESS_USER` Actionがdispatchされるのを待つ
4.  _(誰かが `REQUEST_USER` Actionをdispatchする)_
5.  `call` 作用で `API.user` 関数を呼び出して、通信処理の完了を待つ
6.  _(通信処理が完了する)_
7.  `put` 作用を使って `SUCCESS_USER` Actionをdispatchする
8.  `handleRequestSearchByLocation` タスクが再開して、`call` 作用で `API.searchByLocation` 関数を呼び出して、通信処理の完了を待つ
9.  _(通信処理が完了する)_
10. `put` 作用を使って `SUCCESS_SEARCH_BY_LOCATION` Actionをdispatchする
11. それぞれのタスクにてwhileループの先頭に戻って `take` でActionのdispatchを待つ-->

각각의 태스크를 주의해서 보면 단순한 것 밖에 하지 않기 때문에 이해하기 쉬워지지 않았나요? 게다가 이 코드를 확장하여 체인을 늘리거나, 체인을 순서를 바꾸거나 무엇을 하더라도 Task는 하나만 집중하고 있으므로 다른 무엇을 해도 그다지 영향을 받지 않습니다. 이러한 성질을 적극적으로 이용하여 Task가 너무 비대해지기 전에 계속해서 잘라 나누는 것으로 코드의 건전성을 유지할 수 있습니다.

<!--それぞれのタスクに着目すると単純なことしかしてないので理解しやすいのではないでしょうか。さらにこのコードを拡張してチェインを増やしたり、チェインさせる順番を入れ替えたり、何をするにしてもタスクは１つのことに集中しているので、他がなにしていようとあまり影響を受けません。この性質を積極的に利用してタスクが膨れ上がる前にガンガン切り分けていくとコードの健全性を保てます。-->

### 테스트를 써보자

<!--### テストを書いてみる-->

redux-saga를 적극적으로 쓰고싶은 이유로서 테스트의 편리함을 들었습니다. 아직 다른사람에게 번역할만큼 노하우의 축적이 안되었지만, 분위기를 파악하는 정도로 써보겠습니다. 테스트 대상은 최초의 통신처리의 코드로 합니다. 복잡하게 되기 전의 예제입니다. 먼저 단순해보이는 `rootSaga` Task의 테스트부터 써보겠습니다. 그리고, 테스트코드는 mocha + power-assert 입니다.

<!--redux-sagaを積極的に使いたい理由としてテストのしやすさを挙げました。まだ他人に講釈できるほどのノウハウの蓄積はないのですが、雰囲気をつかむ程度に書いてみます。テスト対象は最初の通信処理のコードにします。複雑にする前の方です。まずは単純そうな `rootSaga` タスクのテストから書いてみます。尚、テストコードは mocha + power-assert です。-->

`sagas.js`

```js
export default function* rootSaga() {
  yield fork(handleRequestUser);
}
```

여기에 대한 테스트 코드는 다음과 같습니다.

<!--これに対するテストコードは次のようになります。-->

`test.js`

```js
describe('rootSaga', () => {
  it('launches handleRequestUser task', () => {
    const saga = rootSaga();

    ret = saga.next();
    assert.deepEqual(ret.value, fork(handleRequestUser));

    ret = saga.next();
    assert(ret.done);
  });
});
```

Task를 포크하고 있는지 테스트를 한다. 라고 하면 어렵게 들리지만, 여기서 Task라는 건 단순한 Generator 함수로, Task가 돌려주는건 모두 단순한 오브젝트라는걸 떠올립시다. 고로 redux-saga에 두어진 Task의 테스트는 단순히 오브젝트를 비교하는 것으로 대부분 충분합니다. 이 `rootSaga` Task가 포크하고 있는지를 확인하기 위해 `fork` Effect로 오브젝트를 생성하여 비교하는 것으로 OK입니다. 이 expected로 지정된 오브젝트도 Task의 작성에 쓰여진 Effect Creator로 생성하여 문제없는것이 재밋는 포인트입니다. **테스트해야 할 것은 이 Task가 무엇을 하고 있는 가로써, 그에 앞서 무엇을 하는가는 알 필요 없습니다.**

<!--タスクをフォークしているかテストする、と言うと難しそうに聞こえますが、ここでタスクというのはただのGenerator関数で、タスクが返すものはすべてただのオブジェクトである、ということを思い出しましょう。つまりredux-sagaにおけるタスクのテストは単純なオブジェクトの比較でほとんど間に合います。この `rootSaga` タスクはフォークしているか調べたいので `fork` 作用でオブジェクトを生成して比較するだけでOKです。このexpectedに指定するオブジェクトもタスクの記述に使われているEffect Creatorで生成して問題ないのも面白いポイントです。 **テストするべきはこのタスクが何をしようとしているかであって、その先で何をするかは知ったこっちゃないわけです。**-->

이것만이면 테스트한 느낌이 안나므로 좀 더 복잡한 `handleRequestUser` Task도 테스트로 써봅니다.

<!--これだけだとテストした気にならないのでもうちょい複雑な `handleRequestUser` タスクの方のテストも書いてみましょう。-->

`sagas.js`

```js
function* handleRequestUser() {
  while (true) {
    const action = yield take(REQUEST_USER);
    const { payload, error } = yield call(API.user, action.payload);
    if (payload && !error) {
      yield put(successUser(payload));
    } else {
      yield put(failureUser(error));
    }
  }
}
```

통신처리가 성공했는지 실패했는지로 분기됩니다. 그러므로 테스트도 각각의 케이스를 적습니다.

<!--通信処理が成功したか失敗したかで分岐します。そのためテストもそれぞれのケースごとに書いてみます。-->

`test.js`

```js
describe('handleRequestUser', () => {
  let saga;
  beforeEach(() => {
    saga = handleRequestUser();
  });

  it('receives fetch request and succeeds to get data', () => {
    let ret = saga.next();
    assert.deepEqual(ret.value, take(REQUEST_USER)); // (A')

    ret = saga.next({ payload: 123 }); // (A)
    assert.deepEqual(ret.value, call(API.user, 123)); // (B')

    ret = saga.next({ payload: 'GOOD' }); // (B)
    assert.deepEqual(ret.value, put(successUser('GOOD')));

    ret = saga.next();
    assert.deepEqual(ret.value, take(REQUEST_USER));
  });

  it('receives fetch request and fails to get data', () => {
    let ret = saga.next();
    assert.deepEqual(ret.value, take(REQUEST_USER));

    ret = saga.next({ payload: 456 });
    assert.deepEqual(ret.value, call(API.user, 456));

    ret = saga.next({ error: 'WRONG' });
    assert.deepEqual(ret.value, put(failureUser('WRONG')));

    ret = saga.next();
    assert.deepEqual(ret.value, take(REQUEST_USER));
  });
});
```

이것은 Generator함수의 테스트가 되므로 익숙하지 않으면 어렵겠네요. 접근하는 방법은 `next()`를 부를때 처음의 `yield`까지 실행되는 그때의 우변의 값을 랩핑한 것이 리턴값으로 나옵니다. 우변 값 자체는 `value` 프로퍼티에 격납되어있으므로 거기서 확인합니다.

<!--これはGenerator関数のテストになるので慣れないうちはややこしいですね。考え方としては `next()` を呼ぶと最初の `yield` まで実行されてそのときの右辺値をラップしたものが戻り値として返ってきます。右辺値自体は `value` プロパティに格納されているのでそれをチェックします。-->

지금, Task는 정지되어있습니다. 이를 재개하기 위해서는 더욱이 `next()`를 불러냅니다. 이 `next()`의 인수로써 넘겨준 것은, Task가 재개될 때에 `yield`로부터 나온 리턴 값이 됩니다. 즉 코드중의 `(A)`로 넘겨지는 것이 `(A')`로부터 나올거라 기대되는 리턴 값이라는 것이죠. 같은 방식으로 `(B)`로 넘겨진 통신결과의 오브젝트가 `(B')`의 `call` Effect로 불러진 결과가 됩니다.

<!--今、タスクは停止しています。これを再開するにはさらに `next()` を呼び出します。この `next()` の引数として渡したものは、タスクが再開したときに `yield` から返ってくる戻り値になります。つまりコード中の `(A)` で渡すものが `(A')` で期待している戻り値、というわけですね。同じように `(B)` で渡した通信結果のオブジェクトが `(B')` の `call` 作用の呼び出し結果になります。-->

마지막으로, 통신처리가 끝나면, 다시 리퀘스트를 기다리는 상태가 되었는지 확인합니다. Task를 동기적으로 썻기에, 테스트 코드도 동기적으로 되었습니다.

<!--最後に、通信処理が終わったら再度リクエストを受け付ける状態になっているか確認しています。タスクを同期的に書いたことで、テストコードも同期的になっています。-->

조금 설명이 빠른 느낌도 있지만, 왜 redux-saga의 실행모델로 제공된 커맨드를 사용하여 Task를 만드는 것인가가 이해되었다고 생각합니다. 모든것이 예측 가능한 테스트가 간단하게 되어, 복잡한 목을 만들 필요성도 최소한으로 되기 때문입니다.

<!--少々駆け足気味でしたが、なぜredux-sagaが用意した実行モデルで提供されたコマンドを使ってタスクを記述するのか理解出来たと思います。すべては予測可能でテストが容易になり、複雑なモックを組み上げる必要性を最小限にするためということです。-->

## 셋업

<!--## セットアップ-->

Task의 설명으로 이것 저것 넘어가버렸기에, 조금더 redux-saga의 설정에 대해 주의점도 알려드리겠습니다. 앞서 말하지만, 기본적인건 공식 문서를 읽는 것이 가장 좋습니다. 이후 대폭 바뀔 가능성은 적지만, 그럴 때 기댈 수 있는건 역시 공식이니까요.

<!--タスクの説明でいろいろとすっとばしてしまったので、ちょっとだけredux-sagaの設定についてハマりポイントと共に書いておきます。前提として、基本的には公式ドキュメントを読むのが一番です。今後大幅に変わる可能性は低そうですが、そうなったときに頼れるのはやはり公式ですしね。-->

### redux-saga의 적용

<!--### redux-sagaを組み込む-->

예제 코드의 디렉토리를 보면 금새 알아차리셨을지 모르겠지만, redux-saga를 쓸 땐 2가지를 합니다. 하나는 Store에 Middleware를 집어넣고, 다른 하나는 Task를 정의합니다. 이하는 전형적인 셋업 코드입니다. `redux-logger`는 필요하지 않으면 지워주세요.

<!--サンプルコードのディレクトリを見てもらった方が手っ取り早いかもしれませんが、redux-sagaを使うときは２つのことをします。１つはStoreへのMiddleware組み込み、そしてもう１つはタスクの定義です。以下は典型的なセットアップのコードになります。`redux-logger` は不要であれば削除してください。-->

`store.js`

```js
import { createStore, applyMiddleware } from 'redux';
import createSagaMiddleware from 'redux-saga';
import logger from 'redux-logger';
import reducer from './reducers';
import rootSaga from './sagas';

export default function configureStore(initialState) {
  const sagaMiddleware = createSagaMiddleware();
  const store = createStore(
    reducer,
    initialState,
    applyMiddleware(
      sagaMiddleware, logger()
    )
  );
  sagaMiddleware.run(rootSaga);
  return store;
};
```

### Store를 초기화시키는 타이밍

<!--### Storeの初期化タイミング-->

이전에 겪었던 일로, 의도하지 않은 페이지에서 redux-saga가 기동되어 통신처리가 시작된 적이 있었습니다. 원인은 `store.js`에 있었습니다.

<!--以前ハマったこととして、意図しないページでredux-sagaが起動して通信処理が始まってしまっていたことがありました。原因は `store.js` で横着していたからでした。-->

`store.js`

```js
const sagaMiddleware = createSagaMiddleware();
const store = createStore(
  reducer,
  applyMiddleware(
    sagaMiddleware, logger()
  )
);
sagaMiddleware.run(rootSaga);
export default store;
```

`configureStore` 함수를 export하는 대신에 작성한 Store를 export하고 있네요. 그리고 `saga.js`는 이런 상태였습니다.

<!--`configureStore` 関数をエクスポートする代わりに作成したStoreをエクスポートしていますね。そして `sagas.js` はこんな感じ。-->

`sagas.js`

```js
export default function* rootSaga() {
  yield fork(loadHogeHoge);
}
```

초기화시 무언가를 읽어들이는 형태의 태스크입니다.

<!--初期化時に何かを読み込むタイプのタスクです。-->

`index.js`

```jsx
import store from './store.js';

// ...

const el = document.getElementById('container');
if (el) {
  ReactDOM.render(
    <Provider store={store}>
      <App />
    </Provider>,
  );
}
```

이미 파악하셨을거라 생각되지만, 위의 구성이면 Provider 컴포넌트가 마운트되었는지 말았는지에 상관없이 Store가 초기화되어 Middleware도 초기화되고 맙니다. 그 결과, 기동시 리퀘스트를 보내는 형태의 Task라면 엉뚱한 타이밍에 사태가 벌어지는 것입니다. 조심하도록 합시다. (저도 반성을..)

<!--もうおわかりと思いますが、以上の構成にするとProviderコンポーネントがマウントされるかどうかに関わらずStoreは初期化されており、Middlewareも初期化されてしまいます。その結果、起動時にリクエストを飛ばすタイプのタスクだと誤爆するというわけです。気を付けましょう（自戒を込めて）。-->

### Middleware의 실행 타이밍

<!--### Middlewareの実行タイミング-->

v0.10.0 부터 redux-saga의 기동방법이 바뀌었습니다.

<!--v0.10.0 からredux-sagaの起動方法が変わりました。-->

`before.js`

```js
const store = createStore(
  reducer,
  applyMiddleware(createSagaMiddleware(rootSaga))
)
```

이렇게 쓰던 것이

<!--こう書いていたのが、-->

`after.js`

```js
const sagaMiddleware = createSagaMiddleware();
const store = createStore(
  reducer,
  initialState,
  applyMiddleware(sagaMiddleware)
);
sagaMiddleware.run(rootSaga);
```

이렇게 되었습니다. 초기실행 Task를 Middleware의 생성시가 아니라, Store의 초기화가 완료된 뒤에 `run` 메소드를 부르는 것으로 기동합니다.

<!--こうなります。初期実行タスクをMiddlewareの作成時ではなく、Storeの初期化が完了したあとに `run` メソッドを呼び出すことで起動します。-->

### 디버그

<!--### デバッグ-->

Task는 하나하나 독립하게 실행되므로, 할 것을 제한해서 단순하게 유지하면 디버그 툴이 필요할 만큼 복잡해지진 않겠지만, 일단, redux-saga에는 모니터링 툴을 집어 넣을 수 있는 인터페이스가 준비되어 있습니다. `effectTriggered`, `effectResolved`, `effectRejected`, `effectCancelled` 의 4개의 프로퍼티를 가지는 오브젝트를 [createSagaMiddleware](http://yelouafi.github.io/redux-saga/docs/api/index.html#createsagamiddlewareoptions) 함수의 오브젝트로써 넘겨줍니다.

<!--１つ１つのタスクは独立して実行されるので、やることを絞って単純に保てばデバッグツールが必要なほど複雑になることは少ないのですが、一応 redux-saga にはモニタリングツールを組み込むためのインターフェイスが用意されています。`effectTriggered`, `effectResolved`, `effectRejected`, `effectCancelled` の４つのプロパティを持つオブジェクトを [createSagaMiddleware](http://yelouafi.github.io/redux-saga/docs/api/index.html#createsagamiddlewareoptions) 関数のオプションとして渡します。-->

`store.js`

```js
import sagaMonitor from './saga-monitor';

export default function configureStore(initialState) {
  const sagaMiddleware = createSagaMiddleware({ sagaMonitor });
  const store = createStore(...
```

모니터의 적용은 일단 [redux-saga의 examples/sagaMonitor](https://github.com/yelouafi/redux-saga/blob/master/examples%2FsagaMonitor%2Findex.js)를 써봐주세요. 그리고, 이 모니터는 디폴트로 아무것도 표시하지 않으므로, 코드중의 `VERBOSE`라는 변소를 `true`로 바꾸셔야 떠들기 시작합니다. 단, `redux-logger`와 같이 로그가 계속해서 흘러가는 걸 보는게 아니라, 필요할 떄 브라우저툴로부터 `window.$$LogSagas` 함수를 불러서 Task tree를 지켜보는게 주 목적 입니다. 실행해보면 다음과 같이 나타납니다. 하지만, 그다지 멋있어 보이진 않으므로 [D3.js](https://d3js.org/)로 시각화 툴을 만들어 볼 생각입니다.

<!--モニターの実装はとりあえず[redux-sagaのexamples/sagaMonitor](https://github.com/yelouafi/redux-saga/blob/master/examples%2FsagaMonitor%2Findex.js)を使ってみてください。尚、このモニターはデフォルトでは何も表示しないので、コード中の `VERBOSE` という変数を `true` にすると騒がしくなります。ただ、`redux-logger` のように常にログが垂れ流されるという使い方ではなくて、必要なときにブラウザの開発者ツールから `window.$$LogSagas` 関数を呼び出してタスクツリーを眺めるのがメインです。実行してみたときの様子は以下です。が、あまりかっこよくないので[D3.js](https://d3js.org/)で可視化するツールを作るつもりです。-->

![saga-monitor.png](https://qiita-image-store.s3.amazonaws.com/0/69860/82c1116f-50b1-8ab1-dc5c-8a1ba9792275.png "saga-monitor.png")

이 다음의 [API 요청시의 스로틀링](http://qiita.com/kuy/items/716affc808ebb3e1e8ac#api%E5%91%BC%E3%81%B3%E5%87%BA%E3%81%97%E3%81%AE%E3%82%B9%E3%83%AD%E3%83%83%E3%83%88%E3%83%AA%E3%83%B3%E3%82%B0)에서 소개하는 예제에는 모니터가 포함되어 있으므로 [데모](http://kuy.github.io/redux-saga-examples/throttle.html)로 시험해 보실 수 있습니다.

<!--このあとの[API呼び出しのスロットリング](http://qiita.com/kuy/items/716affc808ebb3e1e8ac#api%E5%91%BC%E3%81%B3%E5%87%BA%E3%81%97%E3%81%AE%E3%82%B9%E3%83%AD%E3%83%83%E3%83%88%E3%83%AA%E3%83%B3%E3%82%B0)で紹介するサンプルにはモニターが組み込んであるので[デモ](http://kuy.github.io/redux-saga-examples/throttle.html)から試せます。-->

## 실전 redux-saga

<!--## 実践 redux-saga-->

redux-saga에는 [풍부한 예제](https://github.com/yelouafi/redux-saga/tree/master/examples)가 준비되어 있습니다. 뭔가 곤란할 때, 힌트가 없나 싶을 때 보면 좋습니다. ...하지만, 이걸로 마무리하기엔 아쉽기에 다른 이용 예를 소개합니다. 특히 몇일전 릴리즈된 [0.10.0](https://github.com/yelouafi/redux-saga/releases/tag/v0.10.0)의 신기능인 [eventChannel](http://yelouafi.github.io/redux-saga/docs/api/index.html#eventchannelsubscribe-buffer-matcher)를 사용한 예제는 그다지 없으므로 참고가 될 듯 합니다.

<!--redux-sagaには[豊富なサンプル](https://github.com/yelouafi/redux-saga/tree/master/examples)が用意されています。なにか困ったらヒントがないか見てみるといいです。・・・が、これで終わらせてしまうのはあんまりなので別の利用例を紹介します。特に先日リリースされた [0.10.0](https://github.com/yelouafi/redux-saga/releases/tag/v0.10.0) の新機能である [eventChannel](http://yelouafi.github.io/redux-saga/docs/api/index.html#eventchannelsubscribe-buffer-matcher) を使ったサンプルはあまり出回ってないので参考になるかもしれません。-->

> 역자주: 여기서 소개되는 실전 예제들의 소스 코드와 데모들은 밑의 링크에서 찾으실 수 있습니다. 이 포스팅에 설명되지 않은 예제도 있으니 이쪽도 훑어 봐주시면 더욱 좋을 듯합니다.
>
> 소스 코드 : https://github.com/kuy/redux-saga-examples
> 데모 : http://kuy.github.io/redux-saga-examples/

### 자동 완성

<!--### オートコンプリート-->

텍스트 필드에 자동 완성 기능을 넣을 때, 단순하게 만든다면 dispatch된 Action을 `take`로 받아들여 `call`로 요청을 발행하여 결과를 `put`으로 하면 좋아보입니다. 단, 이것은 일반적인 통신처리로써, 그대로 적용해버리면 입력 할 때마다 리퀘스트를 보내기에 그다지 좋지 못합니다. 이 예제에서는 초기의 못생긴 자동 완성 기능을 멋지게 고쳐나가는 과정을 보여드리겠습니다.

<!--テキストフィールドでオートコンプリートを実装するとき、単純にやるならdispatchされたActionを `take` で受け取って `call` でリクエストを発行して結果を `put` すればよさそうです。ただ、これは一般的な通信処理なので、素直に実装すると入力のたびにリクエストが投げられてしまってあまりよろしくありません。このサンプルでは初期のイケてないオートコンプリートからイケてるオートコンプリートに改良していく過程を書いてみます。-->

데모: [Autocomplete](http://kuy.github.io/redux-saga-examples/autocomplete.html)
예제 코드: [kuy/redux-saga-examples > autocomplete](https://github.com/kuy/redux-saga-examples/tree/master/autocomplete)

<!--デモ: [Autocomplete](http://kuy.github.io/redux-saga-examples/autocomplete.html)
サンプルコード: [kuy/redux-saga-examples > autocomplete](https://github.com/kuy/redux-saga-examples/tree/master/autocomplete)-->

#### 초기 구현

<!--#### 初期実装-->

`sagas.js`

```js
function* handleRequestSuggests() {
  while (true) {
    const { payload } = yield take(REQUEST_SUGGEST);
    const { data, error } = yield call(API.suggest, payload);
    if (data && !error) {
      yield put(successSuggest({ data }));
    } else {
      yield put(failureSuggest({ error }));
    }
  }
}

export default function* rootSaga() {
  yield fork(handleRequestSuggests);
}
```

통신처리의 코드 그대로이네요. 참고로 실은 이 코드, 큰 문제를 지니고 있습니다. 무엇인가 하면, 통신처리의 완료를 기다리는 동안 dispatch되는 Action을 흘려보내 버립니다. 예제에서는 통신처리부분을 더미로써 `setTimeout`을 사용하여 시간이 걸리듯이 만들었기 때문에, 이 부분의 시간을 3초 정도로 바꾸면 확실하게 보일겁니다.

<!--通信処理のコードそのままですね。ちなみに実はこのコード、大きな問題を抱えています。なんと、通信処理の完了待ちをしている間にdispatchされたActionを取りこぼします。サンプルでは通信処理部分をダミーにして `setTimeout` を使って時間がかかっているように見せかけているのでその部分の時間を3秒とかに変更してみるとはっきりすると思います。-->

#### 흘림방지 대책

<!--#### 取りこぼし対策-->

이런 이유로 멋지게 만들기 앞서 버그를 잡읍시다. 문제는 `call`로 `API.suggest`의 결과를 기다리는 곳입니다. 이것이 불러지는걸 기다리지 않고 `take`로 돌아가면 흘리지는 않게됩니다. 그렇다면 `fork`로 새로운 Task를 기동시키는 것이 좋아보이네요.

<!--というわけでまずはイケてるオートコンプリートにする前にバグを取りましょう。問題は `call` で `API.suggest` の結果を待つところです。これの呼び出しを待たずに `take` に戻れれば取りこぼしはなくなります。そうすると `fork` で新しくタスクを起動するのがよさそうですね。-->

`sagas.js`

```js
function* runRequestSuggest(text) {
  const { data, error } = yield call(API.suggest, text);
  if (data && !error) {
    yield put(successSuggest({ data }));
  } else {
    yield put(failureSuggest({ error }));
  }
}

function* handleRequestSuggest() {
  while (true) {
    const { payload } = yield take(REQUEST_SUGGEST);
    yield fork(runRequestSuggest, payload);
  }
}

export default function* rootSaga() {
  yield fork(handleRequestSuggest);
}
```

이런 형태가 됩니다. 이것으로 `handleRequestSuggest` Task로 통신처리까지 핸들링하고 있지만, `call` 이후의 부분을 따로 Task로 나누었습니다. 아무리 이번과 같은 문제가 일어나지 않는 다고 해도, Action을 감시하는 Task와 통신처리를 하는 태스크를 나누는 것이 좋아보입니다. 이걸로 막힘없이 리퀘스트도 날릴 수 있네요! 잘 됐네요!

<!--こんな感じになります。これまでは `handleRequestSuggest` タスクで通信処理までハンドリングしていましたが、 `call` 以降の部分を別タスクに分けました。たとえ今回みたいな問題が起きていなかったとしても、Actionを監視するタスクと通信処理をするタスクを分けるというのはよさそうです。これでガンガンリクエストが飛びますね！よかった！-->

#### 다른 해결방법

<!--#### 別の解決方法-->

자, 버그는 고쳤지만 공부를 위해 잠깐 옆길로 새겠습니다. redux-saga로 Task를 쓰고 있으면 위의 패턴이 빈번하게 나오기 때문에 [`takeEvery`](http://yelouafi.github.io/redux-saga/docs/api/index.html#takeeverypattern-saga-args)가 준비되어있습니다. 이것을 사용하여 다시 써봅시다.

<!--さて、バグは直りましたがちょっと勉強のために寄り道します。redux-sagaでタスクを書いていると上記のようなパターンが頻出するため [`takeEvery`](http://yelouafi.github.io/redux-saga/docs/api/index.html#takeeverypattern-saga-args) が用意されています。これを使って書き換えてみましょう。-->

`sagas.js`

```js
import { call, put, fork, takeEvery } from 'redux-saga/effects';

function* runRequestSuggest(action) {
  const { data, error } = yield call(API.suggest, action.payload);
  if (data && !error) {
    yield put(successSuggest({ data }));
  } else {
    yield put(failureSuggest({ error }));
  }
}

function* handleRequestSuggest() {
  yield takeEvery(REQUEST_SUGGEST, runRequestSuggest);
}

export default function* rootSaga() {
  yield fork(handleRequestSuggest);
}
```

`takeEvery` 는 지정한 Action의 dispatch를 기다려, 그 Action을 인수로써 Task를 기동합니다. 이전엔 헬퍼 함수로써 제공되었지만, [0.14.0](https://github.com/redux-saga/redux-saga/releases/tag/v0.14.0)부터 정식으로 Effect가 되었습니다. 단, 헬퍼의 `takeEvery`는 없어질 예정이므로 바꾸실걸 추천합니다. 그리고, Effect로써 `takeEvery`와 헬퍼의 `takeEvery`는 다릅니다. 따라서 Effect인 `takeEvery`를 `yield*`로 사용해서는 안됩니다.

<!--`takeEvery` は指定したActionのdispatchを待って、そのActionを引数としてタスクを起動します。以前はヘルパー関数として提供されていましたが、[0.14.0](https://github.com/redux-saga/redux-saga/releases/tag/v0.14.0) から正式な作用になりました。なお、ヘルパー版の `takeEvery` は廃止予定となっているので移行をおすすめします。また、作用としての `takeEvery` と ヘルパーの `takeEvery` は異なるものです。したがって、作用としての `takeEvery` を `yield*` で使用することはできません。-->

#### 멋진 구현

<!--#### イケてる実装-->

버그도 고쳤고, 이걸로 고칠 준비가 되었습니다. 어떠한 동작이 좋은지 정리하기 위해서 시나리오를 써봅시다.

<!--バグも取れて、これで改善の準備が整いました。どういう動作が望ましいのか整理するためにシナリオを書いてみます。-->

1.  1글자를 입력한다
2.  바로 리퀘스트를 날리지는 않는다.
3.  더 몇문자가 입력된다.
4.  아직 리퀘스트를 보내지 않는다.
5.  아직 입력이 없는 상태가 일정시간 지속되면 리퀘스트를 보낸다.

<!--1.  1文字入力する
2.  すぐにリクエストは投げられない
3.  さらに何文字か入力する
4.  まだリクエストは投げられない
5.  何も入力がない状態が一定時間続くとリクエストが投げられる-->

기본적으로는 일정 시간 기다리면 리퀘스트를 개시하는 지연실행 Task를 정의하여, 입력이 있을 때마다 그것을 기동하게 됩니다. 단, 입력이 있었을때에 이미 지연 Task가 기동되어 있을 때는, 먼저 그것을 취소하고나서 새로운 Task를 기동시킬 필요가 있습니다. 따라서 지연실행 Task는 아무리 많아도 1개만 실행됩니다. 그럼 코드를 봅시다.

<!--基本的には一定時間待ってからリクエストを開始する遅延実行タスクを定義して、入力があるたびにそれを起動することになります。ただし、入力があったときにすでに遅延実行タスクを起動しているときは、まずそれをキャンセルしてから新しいタスクを起動する必要があります。よって遅延実行タスクは最大でも１つしか並行実行されません。それではコードを見てみます。-->

`sagas.js`

```js
import { delay } from 'redux-saga';
import { call, put, fork, take } from 'redux-saga/effects';

function* runRequestSuggest(text) {
  const { data, error } = yield call(API.suggest, text);
  if (data && !error) {
    yield put(successSuggest({ data }));
  } else {
    yield put(failureSuggest({ error }));
  }
}

function forkLater(task, ...args) {
  return fork(function* () {
    yield call(delay, 1000);
    yield fork(task, ...args);
  });
}

function* handleRequestSuggest() {
  let task;
  while (true) {
    const { payload } = yield take(REQUEST_SUGGEST);
    if (task && task.isRunning()) {
      task.cancel();
    }
    task = yield forkLater(runRequestSuggest, payload);
  }
}

export default function* rootSaga() {
  yield fork(handleRequestSuggest);
}
```

주목할 포인트는 2개입니다. 1번째 포인트는 넘겨진 Task를 지연처리하는 `forkLater`함수는 `fork` Effect를 돌려주는 함수입니다. `call` Effect로 `delay` 함수를 불러와 일정시간을 기다리고, `delay` 함수가 돌려주는 Promise가 resolve되면 제어가 돌아와서 Task를 `fork`합니다. 참고로 `delay`함수는 `redux-saga` 모듈로부터 읽어들입니다. 2번째 포인트는 `handleRequestSuggest` Task에 실행중의 지연실행 Task가 있는 경우, 그것을 취소하고나서 기동시키는 부분입니다. `fork` Effect를 `yield` 했을 때 리턴 값은 [Task 인터페이스](http://yelouafi.github.io/redux-saga/docs/api/index.html#task)를 가지는 오브젝트로 기동된 Task의 상태를 가져오거나 취소하는 등, 이것저것 할 수 있습니다.

<!--ポイントは2つあります。１つ目のポイントは渡されたタスクを遅延実行する `forkLater` 関数は `fork` 作用を返す関数です。`call` 作用で `delay` 関数を呼び出して一定時間待ち、`delay` 関数が返すPromiseがresolveされたら制御が戻ってくるのでタスクを `fork` します。ちなみに `delay` 関数は `redux-saga` モジュールからの読み込みです。２つ目のポイントは `handleRequestSuggest` タスクで実行中の遅延実行タスクがあった場合はそれをキャンセルしてから起動する部分です。`fork` 作用を `yield` したときの戻り値は [Taskインターフェイス](http://yelouafi.github.io/redux-saga/docs/api/index.html#task)を実装したオブジェクトで、起動したタスクの状態を取得したりキャンセルしたり、いろいろできます。-->

이러한 방법으로 원하는 동작은 만들어 졌지만, `handleRequestSuggest` Task의 "Action을 기다리는 리퀘스트를 시작한다"라는 역할이 알아보기 어려워졌습니다. 가능한 원래의 Task처럼, 하고 싶은 의도를 전하는 코드라면 좋았겠지요.

<!--この実装で望みの動作は実現できるんですが、`handleRequestSuggest` タスクの「Actionを受け取ってリクエストを開始する」という役割がパッと見て伝わりにくくなっています。できるだけ元のタスクのようにやりたいことの意図が伝わるようなコードだといいですね。-->

`before.js`

```js
function* handleRequestSuggest() {
  while (true) {
    const { payload } = yield take(REQUEST_SUGGEST);
    yield fork(runRequestSuggest, payload);
  }
}
```

자동 완성의 기능만 생각하면 멋지게 되었으니, 이제 코드도 멋지게 만들어 봅시다.

<!--オートコンプリートの機能的にはイケてるので、コードの方もイケてる実装にしてみましょう。-->

#### 더욱 멋진 구현

<!--#### さらにイケてる実装-->

어떻게 하냐면, `handleRequestSuggest` Task에 흩어진 취소를 처리하는 부분을 분리시킵니다. 이는 1개의 Task로 처리하는 일을 줄여 역할을 명확하게 하기 위해 적극적으로 해보고 싶었던 개선 사항입니다.

<!--方針としては `handleRequestSuggest` タスクに散らばってるキャンセル処理の部分を分離します。これは１つのタスクでやることを減らして役割を明確にするという意味で積極的にやっていきたい改善です。-->

`sagas.js`

```js
function* runRequestSuggest(text) {
  const { data, error } = yield call(API.suggest, text);
  if (data && !error) {
    yield put(successSuggest({ data }));
  } else {
    yield put(failureSuggest({ error }));
  }
}

function createLazily(msec = 1000) {
  let ongoing;
  return function* (task, ...args) {
    if (ongoing && ongoing.isRunning()) {
      ongoing.cancel();
    }
    ongoing = yield fork(function* () {
      yield call(delay, msec);
      yield fork(task, ...args);
    });
  }
}

function* handleRequestSuggest() {
  const lazily = createLazily();
  while (true) {
    const { payload } = yield take(REQUEST_SUGGEST);
    yield fork(lazily, runRequestSuggest, payload);
  }
}

export default function* rootSaga() {
  yield fork(handleRequestSuggest);
}
```

`handleRequestSuggest` Task가 매우 깔끔해졌습니다. `fork(runRequestSuggest, payload)`였던 부분이 `fork(lazily, runRequestSuggest, payload)`로 바뀌었을 뿐이기에 변화는 많진 않습니다. 하지만 적어도 영어처럼 "fork lazily"로 읽어지기에 의도를 전달하기가 쉬워졌을 겁니다.

<!--`handleRequestSuggest` タスクがとてもすっきりしました。`fork(runRequestSuggest, payload)` だった部分が `fork(lazily, runRequestSuggest, payload)` に変わるだけなので変化も少ないです。しかも英語っぽく「fork lazily」と読めるので意図も伝わりやすいかもしれません。-->

마법처럼 지연 실행해주는 `lazily` Task이지만 이것은 `createLazily` 함수로 생성하고 있습니다. 실행중의 Task를 존속시키기 위해 클로져로 만들 필요가 있었습니다. 하는 일은 이전의 구현과 동일합니다.

<!--魔法のように遅延実行してくれる `lazily` タスクですがこれは `createLazily` 関数で生成しています。実行中のタスクを保持するためにクロージャにする必要がありました。やっていることは１つ前の実装と同じです。-->

이걸로 기능도 구현도 멋지게 되었습니다.

<!--これで機能も実装もイケてるものになりました！-->

#### 연구과제

<!--#### 研究課題-->

-   지연실행이 시작되기까지 아무것도 표시하지 않는 문제를 해결한다.
-   `takeLatest` 헬퍼함수를 써서 다시 쓴다.

<!---   遅延実行が開始されるまで何も表示されない問題を解決する
-   `takeLatest` ヘルパー関数を使って書き換える-->

### API 요청시의 스로틀링

<!--### API呼び出しのスロットリング-->

데모: [Throttle](http://kuy.github.io/redux-saga-examples/throttle.html)
예제: [kuy/redux-saga-examples > throttle](https://github.com/kuy/redux-saga-examples/tree/master/throttle)

<!--デモ: [Throttle](http://kuy.github.io/redux-saga-examples/throttle.html)
サンプルコード: [kuy/redux-saga-examples > throttle](https://github.com/kuy/redux-saga-examples/tree/master/throttle)-->

포스팅 리스트와 같이 많은 컨텐츠를 한번에 읽어와서, 더욱이 각각의 컨텐츠마다 리퀘스트를 요청하면, 컨텐츠의 수만큼 리퀘스트를 동시에 보내게되니 심각한 일이 일어나겠죠. 서버부하와 같은 문제가 없다고 해도, Dos공격으로 판단되어 리퀘스트가 차단되는 일도 있을 겁니다. 또 통신처리의 경우, 대량발생하는 Action에 따른 Task의 동시 실행 가능한 수 이상은 시작하지 않고 기다리게 만들어서, 앞선 Task가 완료되면 순서대로 다음 Task를 실행시키는 큐(Queue)가 필요할때가 있습니다. 이번 예제는 기동중인 Task의 수를 조절하는 스로틀링을 redux-saga로 구현합니다.

<!--一覧ページなどでたくさんのコンテンツを一気に読み込んで、さらに個々のコンテンツごとにリクエストを開始すると、コンテンツの数だけリクエストが同時に飛んでひどいことになりますね。サーバー負荷的に問題はなかったとしても、DoS攻撃とみなされてリクエストがブロックされる、ということもありえます。また通信処理に限らず、大量発生するActionに付随するタスクを指定した同時実行数以上は起動せずに待たせておき、完了したら順番にタスクを起動するキューが欲しくなることがあります。このサンプルではこういったタスク起動数のスロットリングをredux-sagaで実装しています。-->

`sagas.js`

```js
const newId = (() => {
  let n = 0;
  return () => n++;
})();

function something() {
  return new Promise(resolve => {
    const duration = 1000 + Math.floor(Math.random() * 1500);
    setTimeout(() => {
      resolve({ data: duration });
    }, duration);
  });
}

function* runSomething(text) {
  const { data, error } = yield call(something);
  if (data && !error) {
    yield put(successSomething({ data }));
  } else {
    yield put(failureSomething({ error }));
  }
}

function* withThrottle(job, ...args) {
  const id = newId();
  yield put(newJob({ id, status: 'pending', job, args }));
}

function* handleThrottle() {
  while (true) {
    yield take([NEW_JOB, RUN_JOB, SUCCESS_JOB, FAILURE_JOB, INCREMENT_LIMIT]);
    while (true) {
      const jobs = yield select(throttleSelector.pending);
      if (jobs.length === 0) {
        break; // No pending jobs
      }

      const limit = yield select(throttleSelector.limit);
      const num = yield select(throttleSelector.numOfRunning);
      if (limit <= num) {
        break; // No rooms to run job
      }

      const job = jobs[0];
      const task = yield fork(function* () {
        yield call(job.job, ...job.args);
        yield put(successJob({ id: job.id }));
      });
      yield put(runJob({ id: job.id, task }));
    }
  }
}

function* handleRequestSomething() {
  while (true) {
    yield take(REQUEST_SOMETHING);
    yield fork(withThrottle, runSomething);
  }
}

export default function* rootSaga() {
  yield fork(handleRequestSomething);
  yield fork(handleThrottle);
}
```

자동완성 예제는 1개의 Task만 동시에 실행시키고, 새로운 Task가 오면 처리중인 Task를 취소시키고 나서 기동을 하였습니다. 이미 Task가 기동중인지 아닌지를 판정하기 위해 상태를 가질 필요가 있었고, 그것을 클로져 내에서 다루게 하는 어프로치였습니다. 스로틀링으로도 실행중의 Task를 파악할 필요가 있기 때문에 무언가의 상태를 기다릴 필요가 있는 점이 공통됩니다. 하지만 이번엔 다른 어프로치로, 상태를 Task 내부에서 가지지 않고, 대신 Store에다 넣어둡니다. 이렇게 하면 실행 상태가 뷰에서 실시간 표시가 가능합니다.

<!--オートコンプリートのサンプルは同時実行数は１で、かつ新しいタスクが来たら処理中のタスクをキャンセルしてから起動するという挙動でした。すでにタスクが起動中かどうかを判定するために状態を持つ必要があり、それをクロージャ内に保持するというアプローチです。スロットリングでも実行中のタスクを把握する必要があるため何かしらの状態を持つ必要があるという点で共通しています。せっかくなのでこのサンプルでは別のアプローチをとり、状態をタスク内部に持たず、代わりにStoreに格納します。それによってタスクの実行状況がビュー側にリアルタイムに表示できます。-->

위는 `sagas.js`의 코드만 보여주었지만, 이번엔 상태를 Store가 가지게 하므로 전체를 이해하기 위해 `reducers.js`도 중요하니 한번 훑어봐주세요.

<!--上には `sagas.js` のコードしか示していませんが、今回は状態をStoreに持たせているので `reducers.js` の方も全体を理解する上では大事なので目を通してみてください。-->

#### ２개의 Task

<!--#### ２つのタスク-->

구현은 크게 나누면 2개의 Task, `handleRequestSomething`와 `handleThrottle`로 나뉩니다. 전자는 `REQUEST_SOMETHING` Action의 dispatch를 감시하여 실행해야할 Task만을 보내줍니다. 후자는 조금 복잡합니다. `handleRequestSomething` Task로 부터 실행 요청된 Task를 일단 큐에 넣어두고, 동시 실행 수를 조정하면서 처리해갑니다. 스로틀링이 없는 실행 `fork(runSomething)`과 스로틀링이 있는 실행 `fork(withThrottle, runSomething)`에선 코드의 차이가 조금만 있도록 만들었습니다.

<!--実装は大きくわけて２つのタスク、`handleRequestSomething` と `handleThrottle` に分かれます。前者は `REQUEST_SOMETHING` Actionのdispatchを監視して実行すべきタスクをひたすら投げます。後者はちょっと複雑です。`handleRequestSomething` タスクから実行依頼されたタスクをいったんキューに入れて、同時実行数を調整しながら処理していきます。スロットリングなしの実行 `fork(runSomething)` とスロットリングありの実行 `fork(withThrottle, runSomething)` ではコードの違いはわずかになるよう実装しました。-->

#### ２중 while 루프

<!--#### ２重のwhileループ-->

`handleThrottle` Task를 보면 조금 낯설은 2중의 while 루프가 있습니다. 첫번째 루프는 익숙한 패턴이므로 괜찮을 겁니다. 2번째 루프는 실행가능한 job의 수에 여유가 있는한 job의 실행을 시작 위한 것입니다. 코드의 가독성을 우선해서 while 루프로 만들어져 있지만, 실행 가능한 job의 수와 대기 상태의 job을 만들어 한번에 실행시켜도 괜찮습니다.

<!--`handleThrottle` タスクを見るとちょっと見慣れない２重のwhileループがあります。１つ目のループはおなじみのパターンなので大丈夫ですね。２つ目のループは実行可能なタスク数に空きがある限りジョブの実行を開始するためのものです。コードのわかりやすさを優先してwhileループにしていますが、実行可能なジョブ数と待ち状態のジョブを用意して一気に実行しても大丈夫です。-->

#### 연구과제

<!--#### 研究課題-->

-   다중 실행 큐
-   우선 순위

<!---   複数の実行キュー
-   優先順位-->

### 인증 절차（세션 유지）

<!--### 認証フロー（セッション維持）-->

redux-saga로 인증처리를 어떻게 다루는지를 생각해봅시다.

<!--redux-sagaで認証処理をどう扱うかについて考えてみます。-->

만들고 싶은건, 유저가 로그인하여, 인증받고, 성공하면 화면전이를 하거나, 실패하면 그 이유를 표시하고, 로그아웃하면 다시 대기상태로 돌아가는, 인증 라이프사이클의 전체입니다. 이러한 처리를 서버사이드에 구현하면, Cookie와 같은 토큰을 가지고 날아온 리퀘스트가 어떤 유저로부터 온 건지를 식별할 필요가 있습니다. 즉 처리 자체는 리퀘스트 단위로 되어 토막난 상태입니다. 그것을 redux-saga의(뭐 예제는 혼자서 하므로 식별하는게 그다지 의미없지만) Task가 일시 정지시키는게 가능하다는 특징을 살려서, 인증 라이프사이클 전체를 1개의 Task가 관리하도록 만들어보겠습니다. 한마디로 세션 유지에 Task를 쓰는 형식입니다.

<!--実現したいことは、ユーザーがログインして、認証して、成功したら例えば画面遷移をする、失敗したらその旨を表示する、ログアウトしたらまた待ち受け状態に戻る、というような認証のライフサイクル全体です。こういった処理をサーバーサイドで実装すると、Cookieのようなトークンを持たせて飛んできたリクエストがどのユーザーによるものなのかを識別する必要がありますね。つまり処理自体はリクエスト単位になっていてぶつ切れになります。それをredux-sagaの（まぁ１人しかいないから識別する意味なんてないんだけど）タスクが一時停止可能であるという特徴を活かして、認証のライフサイクル全体を１つのタスクが張り付いて管理するように実装してみます。つまりセッションの維持にタスクを使うイメージです。-->

예제는 어떻게 돌아가는지 분위기 파악을 위한 코드입니다. 원래 [Gist](https://gist.github.com/kuy/18aca23ac885d36eeb5f687a9ad7eff1)에다 쓴 것입니다. 로그인이 성공하면 `react-router-redux`를 사용하여 대시보드 페이지로 이동합니다.

<!--サンプルは雰囲気を掴んでもらうためのコードになります。もともと[Gist](https://gist.github.com/kuy/18aca23ac885d36eeb5f687a9ad7eff1)に書いたものです。ログインが成功すると `react-router-redux` を使ってダッシュボードページに移動します。-->

`sagas.js`

```js
import { push } from 'react-router-redux';

function* authSaga() {
  while (true) {
    // 로그인 할 때 까지 기다린다.
    const { user, pass } = yield take(REQUEST_LOGIN);

    // 인증처리 요청 (여기선 try-catch를 쓰지않고, 리턴 값에 에러정보가 포함되는 스타일）
    const { token, error } = yield call(authorize, user, pass);
    if (!token && error) {
      yield put({ type: FAILURE_LOGIN, payload: error });
      continue; // 인증에 실패하면 재시도와 함께 처음으로 돌아갑니다.
    }

    // 로그인 성공의 처리 (토큰 유지 등)
    yield put({ type: SUCCESS_LOGIN, payload: token });

    // 로그아웃 할 때까지 기다린다.
    yield take(REQUEST_LOGOUT);

    // 로그아웃 처리 (토큰 정리등)
    yield call(SUCCESS_LOGOUT);
  }
}

function* pageSaga() {
  while (true) {
    // 로그인에 성공할 때까지 기다린다.
    yield take(SUCCESS_LOGIN);

    // 대시보드로 이동한다.
    yield put(push('/dashboard'));
  }
}
```

<!--```js
import { push } from 'react-router-redux';

function* authSaga() {
  while (true) {
    // ログインするまでずっと待つ
    const { user, pass } = yield take(REQUEST_LOGIN);

    // 認証処理の呼び出し（ここではtry-catchを使わず戻り値にエラー情報が含まれるスタイル）
    const { token, error } = yield call(authorize, user, pass);
    if (!token && error) {
      yield put({ type: FAILURE_LOGIN, payload: error });
      continue; // 認証に失敗したらリトライに備えて最初に戻る
    }

    // ログイン成功の処理（トークンの保存など）
    yield put({ type: SUCCESS_LOGIN, payload: token });

    // ログアウトするまでずっと待つ
    yield take(REQUEST_LOGOUT);

    // ログアウト処理（トークンのクリアなど）
    yield call(SUCCESS_LOGOUT);
  }
}

function* pageSaga() {
  while (true) {
    // ログイン成功するまでずっと待つ
    yield take(SUCCESS_LOGIN);

    // ダッシュボードページに移動する
    yield put(push('/dashboard'));
  }
}
```-->

#### 2개의 Task

<!--#### ２つのタスク-->

해야 할 일은 인증처리의 라이프사이클을 관리하는 것과, 로그인 성공시에 페이지 전이를 하는 2개입니다. 이것들은 물론 1개의 Task로 구현 가능하지만, redux-saga's way(라는게 있을지는 모르지만)에 따라 제대로 역할별로 Task를 나누어, 각각 `authSaga`와 `pageSaga`로써 정의합니다.

<!--やるべきことは認証処理のライフサイクルの面倒を見ることと、ログイン成功時にページ遷移をすることの２つです。これらはもちろん１つのタスクとして実装できますが、redux-saga's way（というのがあるのかわかりませんが）に従ってきっちり役割ごとにタスクを分けて、それぞれ `authSaga` と `pageSaga` として定義しています。-->

여기까지의 2개의 예제에선 필요에 따라 Task 내부에 외부의 상태를 가지고 있었습니다. 이번 예제는 인증처리의 라이프사이클이 어디까지 되어있나를 상태로 가지는 것을 적극적으로 활용하는 예입니다. 1개의 처리는 1개의 태스크가 항상 붙어있게 되는데 redux-saga가 제공하는 태스크 실행 환경 덕분입니다. 덕분에 코드가 매우 직관적으로 됩니다.

<!--ここまでの２つのサンプルでは必要に迫られてタスク内部または外部に状態を持っていました。このサンプルでは認証処理のライフサイクルがどこまで進んだかというのを状態としてとらえて、それを積極的に活用している例になります。１つの処理に１つのタスクがずっと張り付いていられるのはredux-sagaが提供するタスク実行環境のおかげです。それによってコードがとても直感的になります。-->

#### 연구과제

<!--#### 研究課題-->

-   복수 세션의 유지

<!---   複数セッションの維持-->

### Socket.IO

<!--### Socket.IO-->

여기서부터 조금 다른 종류의 예를 소개하려 합니다. 먼저 [Socket.IO](http://socket.io/)와의 연계입니다.

<!--ここからちょっと変わり種の紹介していきます。まずは[Socket.IO](http://socket.io/)との連携です。-->

예제코드: [kuy/redux-saga-chat-examples](https://github.com/kuy/redux-saga-chat-example)

<!--サンプルコード: [kuy/redux-saga-chat-examples](https://github.com/kuy/redux-saga-chat-example)-->

밑은 예제에서 가져온 내용입니다.

<!--以下、抜粋です。-->

`sagas.js`

```js
function subscribe(socket) {
  return eventChannel(emit => {
    socket.on('users.login', ({ username }) => {
      emit(addUser({ username }));
    });
    socket.on('users.logout', ({ username }) => {
      emit(removeUser({ username }));
    });
    socket.on('messages.new', ({ message }) => {
      emit(newMessage({ message }));
    });
    socket.on('disconnect', e => {
      // TODO: handle
    });
    return () => {};
  });
}

function* read(socket) {
  const channel = yield call(subscribe, socket);
  while (true) {
    const action = yield take(channel);
    yield put(action);
  }
}

function* write(socket) {
  while (true) {
    const { payload } = yield take(`${sendMessage}`);
    socket.emit('message', payload);
  }
}

function* handleIO(socket) {
  yield fork(read, socket);
  yield fork(write, socket);
}

function* flow() {
  while (true) {
    let { payload } = yield take(`${login}`);
    const socket = yield call(connect);
    socket.emit('login', { username: payload.username });

    const task = yield fork(handleIO, socket);

    let action = yield take(`${logout}`);
    yield cancel(task);
    socket.emit('logout');
  }
}

export default function* rootSaga() {
  yield fork(flow);
}
```

Socket.IO으로부터 메세지의 수신에 [eventChannel](http://yelouafi.github.io/redux-saga/docs/api/index.html#eventchannelsubscribe-buffer-matcher)을 사용하고 있습니다. 받은 Socket.IO의 이벤트마다 Redux의 Action으로 맵핑해주어 `put`으로 dispatch하고 있습니다. 거기에 Task의 연쇄적인 취소 역시 쓰여지고 있습니다.

<!--Socket.IOからのメッセージの受信に[eventChannel](http://yelouafi.github.io/redux-saga/docs/api/index.html#eventchannelsubscribe-buffer-matcher)を使っています。受け取ったSocket.IOのイベントごとにReduxのActionにマッピングしてあげて `put` でdispatchしています。さらにタスクの連鎖的なキャンセルも使っています。-->

이 예제는 대충 작동하는지만 돌려본 상태입니다. Read/Write의 부분이나 복수 채널 대응, 하나하나 맵핑이 귀찮아서 이벤트이름을 그대로 Action Types으로 쓰거나, Socket.IO의 통신상태를 모니터링하거나, 어떻게 다룰것인가에 아직 고민중입니다. 언젠가 그것들을 라이브러리로 만들어냈으면 좋겠다라고 생각하고 있습니다.

<!--このサンプルはとりあず動かしてみた、というレベルです。Read/Writeの部分や、複数チャンネル対応、いちいちマッピングが面倒なのでイベント名をそのままAction Typesとして使うとか、Socket.IOの通信状況をモニタリングしたり、どうやって扱うべきなのかまだまだ悩んでいます。いずれそれらをライブラリという形で昇華できたらいいなぁと思ってます。-->

### Firebase (Authentication + Realtime Database)

데모: [Miniblog](http://kuy.github.io/redux-saga-examples/microblog.html)
예제코드 [kuy/redux-saga-examples > microblog](https://github.com/kuy/redux-saga-examples/tree/master/microblog)

<!--デモ: [Miniblog](http://kuy.github.io/redux-saga-examples/microblog.html)
サンプルコード: [kuy/redux-saga-examples > microblog](https://github.com/kuy/redux-saga-examples/tree/master/microblog)-->

[최신 대폭 업데이트](http://jp.techcrunch.com/2016/05/20/20160518google-turns-firebase-into-its-unified-platform-for-mobile-developers/)가 있었던 [Firebase](firebase.google.com)를 써서 시험삼아 redux-saga와 연계시킨 예제입니다. 테마는 트위터와 같은 미니 블로그 서비스이지만, 구현이 덜되서 채팅 앱처럼 되어있습니다... 브라우저의 시크릿모드에 여러개를 열어 각각의 이름으로 투고하면 실시간으로 갱신됩니다.

<!--[最近大幅アップデート](http://jp.techcrunch.com/2016/05/20/20160518google-turns-firebase-into-its-unified-platform-for-mobile-developers/)があった[Firebase](firebase.google.com)を使って試しにredux-sagaで連携させてみたサンプルがこちらになります。題材はTwitterライクなミニブログサービスですが、実装が足りなすぎてただのチャットアプリみたいになってますね・・・。ブラウザのシークレットモードとかで複数開いて別々の名前でログインして投稿するとリアルタイムにタイムラインが更新されます。-->

이러한 예제는 예제 이상의 의미는 없습니다. 어떻게 redux-saga로 Firebase나 Socket.IO를 쓸 것인가에 너무 집착하지는 말아주세요. 왜냐면 redux-saga는 기능적으론 Middleware의 서브셋이므로, redux-saga로 할 수 있는건 Middleware로도 가능합니다. 게다가 redux-saga로 만들면, 프로젝트의 도입할때 redux-saga가 필수가 됩니다. Middleware로 가능한 것을 redux-saga로 만들어서, 도입 장벽을 높이게 되면 의미가 없습니다. 이러한 기능은 순수하게 Middleware나 Store Enhancer 수준으로 구현하는것이 좋지 않을까 싶습니다.

<!--これらのサンプルはサンプル以上の意味はありません。どうかredux-sagaでFirebaseとかSocket.IOを使うべきだ、と捉えないで下さい。なぜかというとredux-sagaは機能的にはMiddlewareのサブセットなので、redux-sagaでできることはMiddlewareでも可能です。さらにredux-sagaで作ってしまうと、プロジェクトに導入するときにredux-sagaが必須になります。Middlewareで可能なものをredux-sagaで作って、導入のハードルを上げる意味はありません。これらの機能は素直にMiddlewareかStore Enhancerレベルで実装してあげるのがいいのではないでしょうか。-->

### PubNub

[redux-saga-chat-example](https://github.com/kuy/redux-saga-chat-example)이라는 redux-saga와 Socket.IO를 합친 채팅 앱을 만드니, 왜인지 [PubNub와 함께 쓸려면 어떻게 해야하지?](https://github.com/kuy/redux-saga-chat-example/issues/2#issuecomment-217758099)라는 질문 있어서 예제를 써봤습니다.

<!--[redux-saga-chat-example](https://github.com/kuy/redux-saga-chat-example)というredux-sagaとSocket.IOを組み合わせたチャットアプリを作ったら、何故か[PubNubと組み合わせるにはどうすればいいの？](https://github.com/kuy/redux-saga-chat-example/issues/2#issuecomment-217758099)という質問が来たのでサンプルコードを書きました。-->

## 만병통치약은 없다

<!--## 銀の弾丸ではない-->

redux-saga의 사용법을 다양한 각도에서 봤습니다. 뭐든 할 수 있을 것 같지만, redux-saga에도 제약이 있습니다. 모든 Middleware의 처리를 이식하는건 불가능 합니다. 예를들어 Middleware처럼 Action을 솎아 내는 건 못합니다. 그래서 이번 샘플은 [Redux의 middleware를 적극적으로 써보자](http://qiita.com/kuy/items/57c6007f3b8a9b267a8e)에서 소개한 [Action을 Dispatch하기 전에 브라우저 확인 다이얼로그를 표시하자](http://qiita.com/kuy/items/57c6007f3b8a9b267a8e#action%E3%82%92dispatch%E3%81%99%E3%82%8B%E5%89%8D%E3%81%AB%E3%83%96%E3%83%A9%E3%82%A6%E3%82%B6%E3%81%AE%E7%A2%BA%E8%AA%8D%E3%83%80%E3%82%A4%E3%82%A2%E3%83%AD%E3%82%B0%E3%82%92%E8%A1%A8%E7%A4%BA%E3%81%99%E3%82%8B)를 그대로 이식하는건 불가능했습니다. 같은 일을 하기엔 우선 다른 Action을 dispatch시켜, 확인 다이럴로그에 Yes가 나오면 원래의 Action을 dispatch하는 식의 변경이 필요했습니다. 이러면 본말전도가 되므로 그냥 Middleware를 쓰는 게 좋은 경우입니다. 덧붙여 이러한 제약은 [Redux Middleware in Depth](http://qiita.com/kuy/items/c6784fe443f1d5c7bbdc)라는 포스팅에도 해설한 Middleware를 실행하는 타이밍 때문에 일어나는 것입니다. redux-saga의 경우, 항상 Reducer의 처리가 끝난 다음에 Saga가 실행되므로, 지금 상태로는 어떻게 할 수 없기 때문입니다. 수요가 있을지는 모르겠지만 redux-saga에 issue를 세워볼까 생각하고 있습니다.

<!--redux-sagaの使い方をいろいろな角度から見てきました。なんでもできそうに思えますが、redux-sagaにも制約はあります。すべてのMiddlewareの処理をそのまま移植できるわけではありません。例えばMiddlewareのようにActionを間引くことはできません。なので今回のサンプルには[Reduxのmiddlewareを積極的に使っていく](http://qiita.com/kuy/items/57c6007f3b8a9b267a8e)で紹介した[ActionをDispatchする前にブラウザの確認ダイアログを表示する](http://qiita.com/kuy/items/57c6007f3b8a9b267a8e#action%E3%82%92dispatch%E3%81%99%E3%82%8B%E5%89%8D%E3%81%AB%E3%83%96%E3%83%A9%E3%82%A6%E3%82%B6%E3%81%AE%E7%A2%BA%E8%AA%8D%E3%83%80%E3%82%A4%E3%82%A2%E3%83%AD%E3%82%B0%E3%82%92%E8%A1%A8%E7%A4%BA%E3%81%99%E3%82%8B)をそのまま移植できませんでした。同じことをやるにはまずは別のActionをdispatchしてもらって、確認ダイアログでYesだったら本物のActionをdispatchするという感じに変更が必要になってしまいます。これだと本末転倒なので素直にMiddlewareを使った方がいいパターンです。ちなみにこの制約は[Redux Middleware in Depth](http://qiita.com/kuy/items/c6784fe443f1d5c7bbdc)という記事で解説したMiddlewareを実行するタイミングに起因するものです。redux-sagaの場合、常にReducerの処理が終わった後にSagaが実行されるため、現状ではどうやっても不可能というわけです。需要があるのかわかりませんが、redux-sagaにissueを立ててみようかなと思っています。-->

## 결론

<!--## まとめ-->

redux-saga를 쓰는 것으로 redux-thunk나 Middleware보다도 구조화된 코드로 비동기처리를 Task라는 실행단위로 기술하는 것이 가능해집니다. 거기에 Mock을 써야하는 테스트를 줄이고, 테스트하고 싶은 로직에 집중 할 수 있습니다. 또한, 재이용가능한 컴포넌트의 개발에서도 필요한 비동기처리를 redux-saga의 Task로서 제공하면, Middleware를 사용하는 경우에 일어나는 실행순서의 문제를 피할 수 있어 안전합니다. 하지만 모든 Middleware를 redux-saga로 바꾸는건 불가능하므로 주의가 필요합니다.

<!--redux-sagaを使うことでredux-thunkやMiddlewareよりも構造化されたコードで非同期処理をタスクという実行単位で記述することができます。さらにモックを使わなければならないテストを減らして、テストしたいロジックに集中できます。また、再利用可能なコンポーネントの開発においても必要な非同期処理をredux-sagaのタスクとして提供することで、Middlewareを使った場合に起こる実行順序の問題を回避できて安全です。それでもすべてのMiddlewareをredux-sagaで置き換えることはできないので注意が必要です。-->
