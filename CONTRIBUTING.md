
# Contributing to the bitcoin-dev blog

1. Find a technical Bitcoin topic you want to write about
  - Most posts are about open-source projects developers work on.
  - If you are not sure if the topic of your post is suitable for this blog, you can get feedback by opening a [topic-review issue].
2. Write the post
  - The contents of a post lives under `content/blog/<blog-post-title>.md`.
  - These files contain a few lines of YAML front matter with, for example, the title and author information.
    - If you have [`hugo`] installed on your system, you can use `hugo new blog/blog-post-title.md` to create a new markdown file from a [template].
    - If you don't have [`hugo`] installed on your system, you can copy an existing blog post, delete the content, and edit the front matter.
  - There is an example blog post showing how to format text and how to embed code, images, and videos. The post is not listed, but can be found [here][example-post] and the [source here][example-source].
3. We strive for high-quality blog posts.
  - The post doesn't have to be perfect but should contain very few formatting, spelling, grammar, or technical mistakes. Try asking other people to review your content first.
  - Your post should arouse interest in the project or topic.
  - We recommended to self-advertise. Include links to your project for interested developers to learn more and to reach out.
  - Keep your blog post roughly between two and six pages. Feel free to split a post up into multiple, independent parts.
4. We follow a few strict rules.
  - Price-talk, altcoins, and politics are off-topic. This is a technical, bitcoin-only blog.
  - Posts you submit should be your original content. You must own the copyright or have permission from the owner to use the material.
  - Posts (text and images) are published under an [Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)] license once merged.
  - Cross-posting from your blog is welcome. Feel free to link to other sites containing the blog post.
5. Open a PR to the [dev-bitcoin/blog repo] and wait for review.



[topic-review issue]: https://github.com/dev-bitcoin/blog/issues/new?template=blog-post-topic-review.md&title=%5BTopic+Review%5D
[`hugo`]: https://gohugo.io/
[template]: https://github.com/dev-bitcoin/blog/blob/main/archetypes/blog.md
[example-post]: https://bitcoin-dev.blog/blog/example-post/
[example-source]: https://github.com/dev-bitcoin/blog/blob/main/content/blog/example-post.md?plain=1
[dev-bitcoin/blog repo]: https://github.com/dev-bitcoin/blog

