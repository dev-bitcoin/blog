---

title: "{{ replace .Name "-" " " | title }}"

# Subtitle (optional)
subtitle: ""

# Date of the blog post. Keep in mind that you won't be able to see future blog post without the "--buildFuture" option in Hugo.
date: {{ .Date }}

# Blog post authors
authors:
  - name: "" # required!
    github: "" # optional
    twitter: "" # optional
  # co-author:
  #- name: "" # required!
  #  github: "" # optional
  #  twitter: "" # optional

# Credit where credit is due. You are encouraged to link to, e.g., your own blog
# where you first published the post. This is optional though.
appearedfirston: # optional
  label: "" # domain of your blog. E.g.: "bitcoin-dev.blog". Without "https://".
  url: "" # full url to your blog post. E.g.: "https://bitcoin-dev.blog/blog/example-post/"

tags:
  #- "Bitcoin Core"
  #- "C++"
  #- "Rust"
  #- "GUI"

# Header image (required!)
# Open-Graph image (og:image) that will be displayed in e.g. the twitter card.
# Also shown on the main page where blog posts are listed.
# Create a directory for your images in the "static/post-data/" folder with
# the title of your blog post.
images:
  - /post-data/<FIXME: folder name here!>/header.png

---


FIXME: A short summary of this post before the 'more'-tag.
Keep it below 100 words.

<!--more-->

FIXME: Your blog post.
See the example post for e.g. ways to include tweets, videos, code, and more.

