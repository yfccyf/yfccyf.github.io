# Yifan Cheng Personal Site

This is the source for `https://yfccyf.github.io`, a GitHub Pages site built with Astro.

## Local Development

```bash
npm install
npm run dev
```

## Build

```bash
npm run build
```

## GitHub Pages Setup

1. Create a public GitHub repository named `yfccyf.github.io`.
2. Push this folder's contents to that repository.
3. In the repository, go to `Settings` -> `Pages`.
4. Under `Build and deployment`, choose `GitHub Actions`.
5. Push to the `main` branch. The workflow in `.github/workflows/deploy.yml` will publish the site.

## Repository Name Note

GitHub user pages must use the account username in the repository name. Because the GitHub account is `yfccyf`, the user page repository should be `yfccyf.github.io`.

To use `yifancheng.github.io`, the GitHub username or organization would need to be `yifancheng`. A custom domain such as `yifancheng.ai`, `yifancheng.dev`, or `yifancheng.com` can later point to this site.
