---
title: "AstroのブログでSvelteを使って目次を作る"
tags:
  - astro
  - svelte
private: false
updated_at: '2026-10-23T01:31:05+09:00'
id: null
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

Astroのブログでサイドバーに見出しを付けたいなと思って、色々読み漁ったけど自家製コンポーネントが一番良いという結論に至った。

# 環境

`package.json`から

- @astrojs/svelte: ^9.1.0
- astro: ^7.4.0
- svelte: ^5.57.2

# コンポーネントを作る

目次の本体はSvelteで書く。他のnpmパッケージとかに依存していないので、たぶん、AIとかに投げればVueとかReactでも使えると思う。

今回はIntersectionObserver APIで見出しの移動を検知して、要素に.currentを付与する。

https://developer.mozilla.org/ja/docs/Web/API/IntersectionObserver

ここでPropsとして受け取ってるheadingsは`render(CollectionEntry): RenderResult`の戻り値。`｛ text: string, slug: string, depth: number ｝`

```svelte:toc.svelte
<script lang="ts">
  import type { MarkdownHeading } from 'astro'
  import { onMount } from 'svelte'

  interface Props {
    headings: MarkdownHeading[]
  }

  let { headings }: Props = $props()

  let last = $state<Element | null>(null)

  // 俺がnoobすぎたのでこれ書き忘れて全然動かなくて死ぬほど悩んだ。良い子はonMountの外に書かないようにしよう。
  onMount(() => {
    const headingElements = document.querySelectorAll('article h2, article h3')
    last = headingElements[0]

    const observer = new IntersectionObserver((entries) =>
      entries.forEach((entry) => {
        if (entry.isIntersecting) last = entry.target
      }),
    )

    headingElements.forEach((heading) => {
      observer.observe(heading)
    })
  })
</script>

<ul class="toc">
  {#each filteredHeadings as { depth, slug, text } (slug)}
    <li class={`${last?.id === slug ? 'current' : ''}`}>
      <a href={`#${slug}`}>{text}</a>
    </li>
  {/each}
</ul>
```

気が向いたらちゃんとdepthを反映しようとは思えど、今のところ不要だから私は書かない。＜ーそもそもこの記事には関係ない

# 呼び出してみる

Astroの記事から呼び出す。

```astro:[...slug].astro
---
import { type CollectionEntry, getCollection } from 'astro:content'
import ToC from './path/to/toc.svelte'

export async function getStaticPaths() {
  const posts = await getCollection('blog')
  return posts.map((post) => ({
    params: { slug: post.id },
    props: post,
  }))
}
type Props = CollectionEntry<'blog'>

const post = Astro.props
const { title } = post.data
const { Content, headings } = await render(post)
---

<div>
  <main>
    <h1>{ title }</h1>
    <Content />
  </main>

  <aside>
    <span>目次</span>
    <ToC headings={headings} client:load />
  </aside>
</div>
```

# おまけ:スタイリングした

![スクリーンショット 2026-10-23 011741.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3342510/f6406df2-65aa-4014-a8c2-57cd63913411.png)

どうでも良いこと:昔ってCSSネストできないから`.scss`に書いてた気がする

```css
ul.toc {
  list-style: none;
  padding: 0;

  display: flex;
  flex-direction: column;

  li {
    border-left: 0.25rem solid #999;
    padding: 0.25rem 0 0.25rem 0.5rem;
    transition:
      0.1s border-left-width,
      0.1s padding-left;

    &.toc-depth-3 {
      padding-left: 2rem;
    }

    &.current {
      border-left-color: #185dcc;
      border-left-width: 0.5rem;
    }

    a {
      color: inherit;
      text-decoration: none;

      &:hover {
        text-decoration: underline;
      }
    }
  }
}
```

# 以上

私は両技術、経験が浅いのでもっといい方法あるかもしれません。
