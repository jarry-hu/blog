# My Blog

This is my personal blog built with Hugo and the PaperMod theme.

## Local Development

1. Clone this repository:
   ```bash
   git clone https://github.com/jarryhum/blog.git
   cd blog
   ```

2. Install Hugo (if not already installed):
   ```bash
   brew install hugo  # macOS
   ```

3. Start the local development server:
   ```bash
   hugo server -D
   ```

4. Open http://localhost:1313 in your browser

## Creating New Posts

To create a new blog post:
```bash
hugo new content posts/my-new-post.md
```

## Deployment

The blog is automatically deployed to GitHub Pages when changes are pushed to the main branch.