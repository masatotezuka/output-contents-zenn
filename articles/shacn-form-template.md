---
title: "shadcn/ui・react-hook-form で汎用性高いフォームコンポーネントを作ってみた"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "reacthookform"
  - "shadcnui"
  - "nextjs"
  - "typescript"
published: true
---

# はじめに

本記事では、shadcn/ui・react-hook-form を使用して汎用性高いフォームのコンポーネント設計について書いています。
shadcn/ui の公式ドキュメント通りに書くと、非常にコードの量が多くなってしまうので、共通化できる部分を切り出してコンポーネントを使い回せるようにしたかったのがモチベーションです。

# 前提

今回使用したフレームワーク/ライブラリです。

- Next.js v15
- zod v3
- react-hook-form v7

# 環境構築

フォームのコンポーネントを作成するまでの準備をしていきます。

1.  pnpm のインストール（パッケージ管理ツールは pnpm でなくても問題ありません）
    公式ドキュメント：https://pnpm.io/ja/installation
    Mac の場合は下記のコマンドでインストールできます。

    ```
    brew install pnpm
    ```

2.  Next.js の環境構築

    ```
    pnpm dlx create-next-app shadcn-form-template
    ```

    ※ node のバージョンを 18.18.0・19.8.0 または 20.0.0 以上にする必要があります

3.  shadcn/ui の環境構築

    ```
    pnpm dlx shadcn@latest init -c ./
    ```

    ※ pnpm の場合は、ディレクトリを指定する必要がありました。

https://github.com/shadcn-ui/ui/issues/5618#issuecomment-2464030449

4.  必要なコンポーネントの追加
    ```
    pnpm dlx shadcn@latest add button form input
    ```
5.  サーバー起動
    ```
    pnpm dev
    ```

# 改善前

公式ドキュメントを参考にフォームのコンポーネントを作成すると、下記のような実装になります。

https://ui.shadcn.com/docs/components/form

```tsx:app/form/page.tsx
"use client"
import { Input } from "@/components/ui/input"
import {
  Form,
  FormControl,
  FormField,
  FormMessage,
  FormLabel,
  FormItem,
} from "@/components/ui/form"
import { Button } from "@/components/ui/button"
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { z } from "zod"

const FormSchema = z.object({
  email: z.string().email({
    message: "メールアドレスの形式が正しくありません",
  }),
  password: z
    .string()
    .min(8, {
      message: "パスワードは8文字以上で入力してください",
    })
    .regex(/(?=.*[0-9])(?=.*[a-z])(?=.*[A-Z])/, {
      message:
        "パスワードは数字・英小文字・英大文字をそれぞれ1文字以上使用してください",
    }),
})

type FormSchemaType = z.infer<typeof FormSchema>

export default function FormPage() {
  const form = useForm<FormSchemaType>({
    defaultValues: {
      email: "",
      password: "",
    },
    resolver: zodResolver(FormSchema),
  })

  const onSubmit = form.handleSubmit((data) => {
    console.log(data)
  })

  return (
    <Form {...form}>
      <form
        onSubmit={onSubmit}
        className="grid grid-cols-1 gap-4 max-w-sm mx-auto mt-6"
      >
        <FormField
          control={form.control}
          name={"email"}
          render={({ field }) => (
            <FormItem>
              <FormLabel>メールアドレス</FormLabel>
              <FormControl>
                <Input
                  {...field}
                  placeholder="メールアドレスを入力してください"
                />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <FormField
          control={form.control}
          name={"password"}
          render={({ field }) => (
            <FormItem>
              <FormLabel>パスワード</FormLabel>
              <FormControl>
                <Input {...field} placeholder="パスワードを入力してください" />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <Button type="submit">送信</Button>
      </form>
    </Form>
  )
}
```

このような書き方は入力項目が増えるたびに、`<FormField/>`,`<FormItem/>`,`<FormControl/>`などフォームを構成するためのコンポーネントを定義する必要があり、コードの見通しが悪くなります。また、コードの書く量も増えるので開発速度が低下します。

# 改善後

このような問題点を改善するためにフォーム用の Input コンポーネントを作成しました。

```tsx:components/form/form-input.tsx
import React from "react"
import { FieldValues, UseControllerProps } from "react-hook-form"
import {
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form"
import { Input, InputProps } from "@/components/ui/input"

export type FormInputProps<T extends FieldValues> = InputProps &
  UseControllerProps<T> & {
    label: string
  }

export function FormInput<S extends FieldValues>({
  name,
  control,
  label,
  ...inputProps
}: FormInputProps<S>) {
  return (
    <FormField
      control={control}
      name={name}
      render={({ field }) => (
        <FormItem>
          <FormLabel>{label}</FormLabel>
          <FormControl>
            <Input
              {...inputProps}
              onChange={field.onChange}
              value={field.value}
              onBlur={field.onBlur}
              disabled={field.disabled}
              name={field.name}
              ref={field.ref}
            />
          </FormControl>
          <FormMessage />
        </FormItem>
      )}
    />
  )
}
```

## ポイント

`form.control`と`name`を受け取れようにするために、型エイリアスにジェネリクスを使用した`UseControllerProps<S>`を宣言する必要があります。(上記のコードは、ジェネティクスの部分を区別するために `S` と`T` で書き分けています。)

さらに、この`S`は`react-hook-form`側で宣言されている`FieldValues`という型エイリアスを継承しているので、自前で定義する時も`FieldValues`を継承しないと型エラーが出ます。

https://github.com/react-hook-form/react-hook-form/blob/5db95c93b701ec98ec43ffd8b04efe8016b928dd/src/types/controller.ts#L35-L49

ちなみに`FieldValues`の実態は、`Record<string, any>`なので、これをそのまま継承しても型エラーは出ません。（any を書くので lint に怒られると思いますが）
https://github.com/react-hook-form/react-hook-form/blob/5db95c93b701ec98ec43ffd8b04efe8016b928dd/src/types/fields.ts#L26

## 改善後のフォームのコンポーネント

フォーム側は`FormInput`を呼び出せばいいので、コードがだいぶスッキリしました。

```tsx:app/form/page.tsx
"use client"
import { Form } from "@/components/ui/form"
import { Button } from "@/components/ui/button"
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { z } from "zod"
import { FormInput } from "@/components/form/form-input"

const FormSchema = z.object({
  email: z.string().email({
    message: "メールアドレスの形式が正しくありません",
  }),
  password: z
    .string()
    .min(8, {
      message: "パスワードは8文字以上で入力してください",
    })
    .regex(/(?=.*[0-9])(?=.*[a-z])(?=.*[A-Z])/, {
      message:
        "パスワードは数字・英小文字・英大文字をそれぞれ1文字以上使用してください",
    }),
})

type FormSchemaType = z.infer<typeof FormSchema>

export default function FormPage() {
  const form = useForm<FormSchemaType>({
    defaultValues: {
      email: "",
      password: "",
    },
    resolver: zodResolver(FormSchema),
  })

  const onSubmit = form.handleSubmit((data) => {
    console.log(data)
  })

  return (
    <Form {...form}>
      <form
        onSubmit={onSubmit}
        className="grid grid-cols-1 gap-4 max-w-sm mx-auto mt-6"
      >
        <FormInput
          control={form.control}
          name="email"
          label="メールアドレス"
          placeholder="メールアドレスを入力してください"
        />
        <FormInput
          control={form.control}
          name="password"
          label="パスワード"
          placeholder="パスワードを入力してください"
        />
        <Button type="submit">送信</Button>
      </form>
    </Form>
  )
}
```

以上が、shadcn/ui・react-hook-form を使用して汎用性高いフォームのコンポーネント設計についてになります。
コンポーネントの再利用性を意識してどこを共通化するかが難しいですが、今回のコンポーネント設計は再利用性高くできたかなと思います。
もっといい設計案があれば、ぜひ教えてください。

参考までにリポジトリを添付します。
https://github.com/masatotezuka/shadcn-form-template

最後までお読みいただきありがとうございました。

# 参考

https://zenn.dev/manalink_dev/articles/manalink-react-hook-form-v7#%E3%83%AD%E3%82%B8%E3%83%83%E3%82%AF%E5%B1%A4

https://nextjs.org/

https://react-hook-form.com/

https://ui.shadcn.com/
