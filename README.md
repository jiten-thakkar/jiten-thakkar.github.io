# Jiten Thakkar's Blog

Personal blog built with [Hugo](https://gohugo.io/) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme.

## Local Development

1. Install Hugo:
   ```bash
   brew install hugo
   ```

2. Clone the repository with submodules:
   ```bash
   git clone --recurse-submodules https://github.com/jiten-thakkar/jiten-thakkar.github.io.git
   ```

3. Run the development server:
   ```bash
   hugo server -D
   ```

4. Visit `http://localhost:1313` to see the site

## Creating New Posts

```bash
hugo new posts/my-new-post.md
```

## Photo Gallery

To add a photo gallery to a post, use the gallery shortcode:

```markdown
{{< gallery pattern="images/*" >}}
```

Place your images in the post's page bundle directory.

## Deployment

The site automatically deploys to GitHub Pages when changes are pushed to the main branch via GitHub Actions.

## License

Content is copyrighted. Theme is licensed under MIT.
