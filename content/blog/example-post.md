---

title: "This is an example post"
subtitle: "Showcasing how to write your own bitcoin-dev.blog post."
date: 2021-10-20T00:00:00+00:00

# This isn't really needed for this blog post, but should show-case
# that you can and are encuraged to link to, e.g., your own blog
# where the post was first published.
appearedfirston:
  label: bitcoin-dev.blog
  url: "https://bitcoin-dev.blog/blog/example-post/"

authors:
  - name: "0xB10C"
    github: "0xb10c"
    twitter: "twitter"
  - name: "Co-Author-1"
    github: "bitcoin"
    twitter: "bitcoincoreorg"
images:
  - "/post-data/example-post/header.png"

# enables MathJax support for this blog post
# Only set this if you are using MathJax in your post.
mathjax: true

# never list this example on the blog
_build:
  list: never

---

Here is the description of the example blog post.
You'd want to summarize the contents of the following post.
Don't let this get too long, though!

<!--more-->



Here the blog post starts.
You can find the [source] on GitHub.
Please read the [CONTRIBUTING.md] before you start writing a post for this blog.

[CONTRIBUTING.md]: https://github.com/dev-bitcoin/blog/blob/main/CONTRIBUTING.md
[source]: https://github.com/dev-bitcoin/blog/blob/main/content/blog/example-post.md?plain=1

#### Text

You can use markdown formatting.

- **Bold text.**
- _Italic text._
- *More italic text.*
- ~~Strikethrough text.~~


#### Links

Markdown lets us include links in three formats.
[Inline-style links](https://your-looooong-url.btc) can often get very long and make reading the markdown source harder.
[Reference-style links] are shorter and thus preferred.

[Reference-style links]: https://your-looooong-url.btc

#### Images

For images, we could use markdown image tags too.
However, it's recommended to use the custom `center-figure` [Hugo shortcode].
This is based on the built-in [`figure`] shortcode but automatically centers the image and captions.

[Hugo shortcode]: https://gohugo.io/content-management/shortcodes/
[`figure`]: https://gohugo.io/content-management/shortcodes/#figure

```
{{</* center-figure src="/img/logo-512.png" caption="The bitcoin-dev.blog logo." height=128 width=128 */>}}
```
produces

{{< center-figure src="/img/logo-512.png" caption="The bitcoin-dev.blog logo." height=128 width=128 >}}

Generally, including images in the repository is preferred to linking to external images.
Check if you have permission to include the image.

#### Code

The built-in shortcode [`highlight`] can be used to showcase code.

```
{{</* highlight rust */>}}
  let mut builder = wallet_bob.build_tx();
  builder
    .add_recipient(addr.script_pubkey(), 60_000)
    .add_foreign_utxo(alice_outpoint, alice_psbt_input, satisfaction_weight)?;
{{</* /highlight */>}}
```

produces


{{< highlight rust >}}
  let mut builder = wallet_bob.build_tx();
  builder
    .add_recipient(addr.script_pubkey(), 60_000)
    .add_foreign_utxo(alice_outpoint, alice_psbt_input, satisfaction_weight)?;
{{< /highlight >}}


[`highlight`]: https://gohugo.io/content-management/shortcodes/#highlight

#### Footnotes

We can include footnotes[^this-is-a-footnote-ref] by using `[^ref]` where `ref` is reference to a `[^ref]: <footnote-text>`.

[^this-is-a-footnote-ref]: This is a footnote.


#### Tweets

Hugo offers a built-in tweet shortcode too.

```
{{</* tweet user="halfin" id=1110302988 */>}}
```

{{< tweet user="halfin" id=1110302988 >}}


#### Videos

Vimeo and YouTube videos can be embedded with the [`vimeo`] and [`youtube`] tags.

[`vimeo`]: https://gohugo.io/content-management/shortcodes/#vimeo

[`youtube`]: https://gohugo.io/content-management/shortcodes/#youtube

`{{</* vimeo 412058509 */>}}` produces
{{< vimeo 412058509 >}}


`{{</* youtube Ey0xAcN11zk */>}}`
produces


{{< youtube Ey0xAcN11zk >}}

---

#### $\LaTeX$ via MathJax

$\LaTeX$ formulars can be embeeded with [MathJax](https://www.mathjax.org/).
MathJax is disabled by default, but can be enabled by setting `mathjax: true` in the front-matter.
See the front-matter of this post for an example.

In-line LaTeX can be written as $1 + 1 = 3$. This also works in headings.
Due to Hugo's MarkDown parser, multi-line formulars [need six backslashes](https://github.com/wowchemy/wowchemy-hugo-themes/issues/291#issuecomment-334746889) (`\\\\\\\\\\\\`) between new lines.
See the example below.

$$
\begin{align}
p & : \text{private key}  \\\\\\
P = pG & : \text{public key}  \\\\\\
m & : \text{message}  \\\\\\
r & : \text{random nonce}  \\\\\\
R = rG & : \text{nonce commitment}
\end{align}
$$
