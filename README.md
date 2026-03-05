# Welcome to my portfolio!

Since I always forget how to manage it, I'll record here different useful commands. They might be useful for you if you're curious or want to reproduce my site.

## Spawning the portfolio locally

Hugo can be deployed locally:

```bash
hugo server
```

You may also deploy drafts for WIP:

```bash
hugo server -D
```

The previous command will fail to link the pages correctly due to the mismatch in the base URL. Instead, use:

```bash
hugo server -D --baseURL http://localhost:1313/
```

Creating a new post is as simple as adding a new entry to `content/posts/`. However, sometimes I prefer the fancy cli way:

```bash
hugo new content content/posts/my-great-post.md
```


