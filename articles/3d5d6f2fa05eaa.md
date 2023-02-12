---
title: "Promise.allSettledよりもPromise.allを使ったほうがよかった話"
emoji: "🙆‍♀️"
type: "tech"
topics:
  - "nestjs"
  - "express"
  - "ts"
published: true
published_at: "2022-10-16 17:17"
---

# 背景
バッチ処理で複数ユーザーにメール配信する場面があったのですが、`Promise.all()`と`Promise.allSettled`の挙動の理解に時間がかかったので、メール配信することを想定して記事にまとめてみました。

# 実現したいこと
- 複数ユーザーにメール配信する処理おいて、一部のユーザーにエラーが起きても、エラーが起きていないユーザーにはメール配信をする
- エラーが起きたとき、配信ができなかったユーザーを特定してエラー内容を通知する

# Promise.allとPromise.allSettledの違い
前提として、Promise.allとPromise.allSettledの違いを整理します。Promise.allとPromise.allSettledは両方とも、**Promiseオブジェクトの配列**を引数として受け取り、新しいPromiseオブジェクトの配列を返します。

Promise.allは、エラーが発生したら拒否（reject）されて、最初に発生したrejectの値を返します。
>Promise.all() メソッドは入力としてプロミスの集合の反復可能オブジェクトを取り、入力したプロミスの集合の結果の配列に解決される単一の Promise を返します。この返却されたプロミスは、入力したプロミスがすべて解決されるか、入力した反復可能オブジェクトにプロミスが含まれていない場合に解決されます。入力したプロミスのいずれかが拒否されるか、**プロミス以外のものがエラーを発生させると直ちに拒否され、最初に拒否されたメッセージまたはエラーをもって拒否されます。** 
>[公式ドキュメントより引用](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)

一方で、Promise.allSettledはエラーが起きても、必ず成功のオブジェクトを返します。
>Promise.allSettled() メソッドは、与えられたすべてのプロミスが履行されたか拒否された後に、**それぞれのプロミスの結果を記述した配列オブジェクトで解決されるプロミスを返します。**
>[公式ドキュメントより引用](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled)

Promise.allSettledの返り値は配列ですが、成功と失敗かどうかで配列の各要素は決まります。成功の場合は`{status:"fulfilled",value:結果の値}`で、失敗の場合は`{status:"rejected",value:結果の値（エラーの内容）}`というオブジェクトが返されます。
このことから、Promise.allSettledではエラーが起きても必ず実行されるのでエラーをキャッチする必要はないと考えまています。

# 実装
上記の内容を踏まえて、実装していきます。
## 準備
まず、メール送信する関数は仮想で作成します。メールアドレスが取得できなかった（emailがnull）場合、エラーが起こるようにします。
```ts
function sendMail(email: string | null): Promise<"SUCCESS" | "REJECTED"> {
  console.log(email);
  return new Promise((resolve, reject) => {
    if (email) {
      resolve("SUCCESS");
    } else {
      reject(new Error("REJECTED"));
    }
  });
}
```

## Promise.allSettledでの実装
エラーが起きてもすべての処理は実行させたいので、今回実現したいことから考えると`Promise.allSettled`を使用するほうが適切かと思います。したがって、まずはPromise.allSettledで実装します。

```ts
Promise.allSettled(
  [
    { id: 1, email: "test1@test.com" },
    { id: 2, email: null },
    { id: 3, email: "test2@test.com" },
  ].map(async (user) => {
    const results = await sendMail(user.email);
    //sendMail()内で出力されたログ
    //test1@test.com
    //null
    //test2@test.com
    return results;
  })
).then((results) => {
  //メール配信結果のログ
  console.log(results);
  //0: {status: 'fulfilled', value: 'SUCCESS'}
  //1: {status: 'rejected', reason: Error: REJECTED at http://localhost:3000/index.js:8:20 at new Promise (<anonymous>) at …}
  //2: {status: 'fulfilled', value: 'SUCCESS'}

  //エラーが起きた時の通知
  results.forEach((result) => {
    if (result.status === "rejected") console.log(result.reason);
    //Error: REJECTED
  });
});
```

## Promise.allでの実装
前述の通り、`Promise.all`ではエラーが発生したら拒否されるのですが、`sendMail()`でエラーをキャッチすれば`Promise.all`でもすべて処理を実行することができます。

```ts
Promise.all(
  [
    { id: 1, email: "test1@test.com" },
    { id: 2, email: null },
    { id: 3, email: "test2@test.com" },
  ].map(async (user) => {
    return await sendMail(user.email).catch((error) => {
    //sendMail()内で出力されたログ
    //test1@test.com
    //null
    //test2@test.com
    
      //エラーが起きたときのログ
      console.log(`REJECTED_USER_${user.id}`);
      //REJECTED_USER_2
      console.log(error);
      //Error: REJECTED
      throw error;
    });
  })
)
  .then((result) => {
    //メール配信結果のログ（エラーが起きていない場合）
    console.log(result);
  })
  .catch((error) => {
    //メール配信結果のログ（エラーが起きた場合）
    console.log(error);
    //Error: REJECTED
  });
```

# まとめ
最初、Promise.allSettledを使用したほうがいいと考えていました。しかし、Promise.allSettledで拒否されたときに`{status:"rejected",value:結果の値（エラーの内容）}`というオブジェクトが返されるので、メール配信されなかったユーザーを特定できないことが問題でした。
Promise.allでもsendMailでエラーをキャッチすれば、すべての処理を実行できることに気付いたので今回のケースではPromise.allを使用したほうがいいという結論に至りました。
もし、他にもいい実装方法があればぜひ教えてください。
また、誤った表記があればご指摘いただけると幸いです。