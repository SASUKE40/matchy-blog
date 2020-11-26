# Matchy's Blog

Based on [Hexo](https://hexo.io/). Used [Fluid](https://github.com/fluid-dev/hexo-theme-fluid) theme.

## How To Use

Refer to the [Fluid Theme User Guide](https://fluid-dev.github.io/hexo-fluid-docs/guide) for more information.

### Create New Post

```bash
hexo new post "example post"
```

The command will create `example-post.md` and place it in `/source/_posts/` directory.

### Create New Draft

```bash
hexo new draft "example draft"
```

The command will create `example-draft.md` and place it in `/source/_drafts/` directory.

### Frontmatter

- `title`
- `tags`
- `excerpt`: manually set excerpt
- `index_img`: the thumbnail image shown to the right of title and except on Home page; can be images in `/img/` or external link
- `banner_img`: the banner image shown at the top of the post page; can be images in `/img/` or external link

## Plug-ins used

- `patch-package`
- `hexo-deployer-git`
- `hexo-douban` (patched)
- `hexo-all-minifier`
