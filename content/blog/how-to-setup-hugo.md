---
date: '2026-04-12T14:40:01+08:00'
draft: false
title: 'How to Build and Deploy a Static Blog with Hugo, GitHub, and Cloudflare Pages'
tags: ["hugo", "cloudflare", "github", "tutorial"]
summary: "A step-by-step guide to building a fast personal blog with the PaperMod theme, deploying it on Cloudflare Pages, and connecting your own custom domain — all for free."
showtoc: true
---

Static sites are fast, cheap to host, and simple to maintain. In this guide you will build a personal blog using Hugo (a static site generator), the PaperMod theme, GitHub for version control, and Cloudflare Pages for free global hosting with automatic deployments. By the end, every time you push to GitHub your site will rebuild and go live within seconds.

## What You Will Build

- A clean, fast personal blog with Blog, About Me, and Links pages
- Automatic deployment on every `git push`
- Free HTTPS, global CDN, and a custom domain

---
<br>
## Phase 1: Install Everything You Need

You need four things: Git, Hugo (extended edition), a text editor, and a GitHub account.
<br>

### 1.1 Update Your System
<br>

Open a terminal and run:

```bash
sudo apt update && sudo apt upgrade -y
```
<br>

### 1.2 Install Git

```bash
sudo apt install git -y
git --version   # should print "git version 2.x.x"
```

Then configure your identity:

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```
<br>


### 1.3 Install Hugo (Extended Edition)

PaperMod requires the **extended** version of Hugo for style processing.

**Option A — Snap (easiest):**

```bash
sudo snap install hugo
hugo version   # confirm output contains "extended"
```
<br>

**Option B — Official release (recommended for latest version):**

Go to [github.com/gohugoio/hugo/releases](https://github.com/gohugoio/hugo/releases) and download the `.deb` file with "extended" and "linux-amd64" in the name, then install it:

```bash
# Replace the version number with the latest available
wget https://github.com/gohugoio/hugo/releases/download/v0.147.8/hugo_extended_0.147.8_linux-amd64.deb
sudo dpkg -i hugo_extended_0.147.8_linux-amd64.deb
hugo version
```

> **Note:** Make a note of your Hugo version number (e.g. `0.147.8`). You will need it later when configuring Cloudflare Pages.
<br>

<br>


### 1.4 Install a Text Editor

VS Code works well and is free:

```bash
sudo snap install code --classic
```

Any editor works — even the built-in Ubuntu text editor is fine for editing config files.

<br>

### 1.5 Create a GitHub Account

Go to [github.com](https://github.com), sign up, and verify your email. GitHub is where your site's source files will live; Cloudflare Pages reads from it to build and publish your site.

<br>

---

## Phase 2: Create Your Hugo Site
<br>

```bash
hugo new site my-blog --format yaml
cd my-blog
```

This creates a folder called `my-blog` with Hugo's default structure and uses YAML for configuration, which PaperMod expects. Then initialise Git:

```bash
git init
```
<br>

---
<br>

## Phase 3: Install the PaperMod Theme

Install PaperMod as a Git submodule — this links to the theme's repository rather than copying it, making updates easy and working seamlessly with Cloudflare Pages:

```bash
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

> If you ever clone your repo to a different machine, run `git submodule update --init --recursive` to restore the theme.

---
<br>

## Phase 4: Configure Your Site

Open `hugo.yaml` in your editor and replace its contents with the following. Read the comments to understand what each section does:

```yaml
baseURL: "https://yourname.com/"
title: Your Site Name
paginate: 5
theme: PaperMod

enableRobotsTXT: true
buildDrafts: false
buildFuture: false
buildExpired: false

minify:
  disableXML: true
  minifyOutput: true

params:
  env: production
  title: Your Site Name
  description: "A brief description of your site"
  keywords: [Blog, Portfolio, Personal]
  author: Your Name
  DateFormat: "January 2, 2006"
  defaultTheme: auto   # auto, light, or dark
  disableThemeToggle: false
  ShowReadingTime: true
  ShowShareButtons: false
  ShowPostNavLinks: true
  ShowBreadCrumbs: true
  ShowCodeCopyButtons: true
  ShowWordCount: false
  ShowRssButtonInSectionTermList: true
  UseHugoToc: true
  disableSpecial1stPost: false
  disableScrollToTop: false
  comments: false
  hidemeta: false
  hideSummary: false
  showtoc: true
  tocopen: false

  # Home page greeting
  homeInfoParams:
    Title: "Hi there!"
    Content: >
      Welcome to my blog. I write about things
      that interest me. Feel free to look around.

  # Social icons shown on the home page
  socialIcons:
    - name: github
      url: "https://github.com/yourusername"
    - name: twitter
      url: "https://twitter.com/yourusername"
    - name: linkedin
      url: "https://linkedin.com/in/yourusername"
    - name: email
      url: "mailto:you@example.com"

# Navigation menu
menu:
  main:
    - identifier: blog
      name: Blog
      url: /blog/
      weight: 10
    - identifier: about
      name: About Me
      url: /about/
      weight: 20
    - identifier: links
      name: Links
      url: /links/
      weight: 30

# Output formats needed for search and RSS
outputs:
  home:
    - HTML
    - RSS
    - JSON
```

Go through and replace every placeholder (`Your Name`, `yourname.com`, social URLs) with your real details. Delete any social icons you don't use.

> **Tip:** YAML is sensitive to indentation. Always use spaces (not tabs) and keep alignment exactly as shown. If something breaks, check indentation first.
<br>

---
<br>

## Phase 5: Create Your Content Pages
<br>

### 5.1 Blog Section

```bash
mkdir -p content/blog
```

Create `content/blog/_index.md`:

```markdown
---
title: "Blog"
description: "All my blog posts"
---
```
<br>

### 5.2 Your First Post

```bash
hugo new blog/my-first-post.md
```

Open `content/blog/my-first-post.md`. Change `draft: true` to `draft: false` when you are ready to publish, and add your content in Markdown below the closing `---`:

```markdown
---
title: "My First Post"
date: 2026-04-12T10:00:00+08:00
draft: false
tags: ["introduction", "hello"]
summary: "A short description of this post"
showtoc: true
---

## Welcome

This is my first post. Here is what I plan to write about.

## A Quick Markdown Primer

You can use **bold**, *italic*, and [links](https://example.com).

Code blocks look like this:

```python
print("Hello, World!")
```
```

### 5.3 About Me Page

Create `content/about.md` and fill it in with your background and interests:

```markdown
---
title: "About Me"
url: "/about/"
summary: "about"
hidemeta: true
ShowBreadCrumbs: false
---

## Who I Am

A few paragraphs about yourself.

## What I Do

Your work, interests, or background.
```
<br>

### 5.4 Links Page

Create `content/links.md`:

```markdown
---
title: "Links"
url: "/links/"
summary: "links"
hidemeta: true
ShowBreadCrumbs: false
---

## Social Platforms

| Platform | Link |
|----------|------|
| GitHub   | [yourusername](https://github.com/yourusername) |
| Twitter  | [@yourusername](https://twitter.com/yourusername) |
| LinkedIn | [Your Name](https://linkedin.com/in/yourusername) |

## Content Feeds

| Type | Link |
|------|------|
| Blog | [/blog](/blog/) |
| RSS  | [RSS Feed](/index.xml) |
```
<br>

---
<br>

## Phase 6: Preview Locally

```bash
hugo server -D
```

The `-D` flag includes draft posts. Open [http://localhost:1313](http://localhost:1313) in your browser. Any file change you save will instantly reload the page. Press `Ctrl+C` to stop the server.

Before moving on, verify:

- The home page shows your greeting and social icons
- The Blog, About Me, and Links menu items all work
- The dark/light theme toggle works

---
<br>

## Phase 7: Push to GitHub
<br>

### 7.1 Create a Repository

Go to [github.com/new](https://github.com/new), name your repository (e.g. `my-blog`), leave everything else as default, and click **Create repository**. Do not check "Add a README file."
<br>

### 7.2 Create a .gitignore

Create `.gitignore` in your project root:

```
public/
resources/
.hugo_build.lock
```
<br>

### 7.3 Commit and Push

```bash
git add .
git commit -m "Initial commit: Hugo site with PaperMod"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/my-blog.git
git push -u origin main
```

Replace `YOUR-USERNAME` and `my-blog` with your actual GitHub username and repository name.

If you have two-factor authentication enabled, you will need a **Personal Access Token** instead of your password. Generate one at [github.com/settings/tokens](https://github.com/settings/tokens) — give it the `repo` scope and use the token as your password when prompted.
<br>

---
<br>

## Phase 8: Deploy on Cloudflare Pages
<br>

### 8.1 Create a Cloudflare Account

Sign up at [dash.cloudflare.com/sign-up](https://dash.cloudflare.com/sign-up) and verify your email.
<br>

### 8.2 Create a Pages Project

1. Log in to the Cloudflare dashboard
2. Go to **Workers & Pages** in the left sidebar
3. Click **Create** → **Pages** tab → **Connect to Git**
4. Authorise Cloudflare to access your GitHub account
5. Select your repository and click **Begin setup**
<br>

### 8.3 Configure Build Settings

| Setting | Value |
|---------|-------|
| Production branch | `main` |
| Framework preset | Hugo |
| Build command | `hugo` |
| Build output directory | `public` |
<br>

### 8.4 Set the Hugo Version (Critical)

Cloudflare's default Hugo version is outdated and will likely fail to build your site. You must pin your version:

1. In the setup page, expand **Environment variables (advanced)**
2. Click **Add variable**
3. Name: `HUGO_VERSION`, Value: your version number (e.g. `0.147.8`)

This must match the version on your local machine — run `hugo version` to confirm.
<br>

### 8.5 Deploy

Click **Save and Deploy**. Wait for the build to complete (usually under a minute). Your site is now live at a `*.pages.dev` URL.

From this point on, every `git push` to `main` triggers an automatic rebuild.
<br>

---
<br>

## Phase 9: Connect a Custom Domain

If you have a domain registered with a registrar (Squarespace, Namecheap, etc.), you can point it to your Cloudflare Pages site.
<br>

### 9.1 Add Your Domain to Cloudflare

1. In the Cloudflare dashboard, click **Websites** → **Add a site**
2. Enter your domain and select the **Free** plan
3. Cloudflare will scan your existing DNS records — review and continue
4. Cloudflare will assign you two nameservers (e.g. `ada.ns.cloudflare.com`)
<br>

### 9.2 Update Nameservers at Your Registrar

Log in to your domain registrar and replace the existing nameservers with the two Cloudflare nameservers shown in the previous step. This change can take up to 48 hours to propagate, but usually completes within an hour.
<br>

### 9.3 Add the Custom Domain to Your Pages Project

Once Cloudflare confirms your domain is active:

1. Go to **Workers & Pages** → your project → **Custom domains**
2. Click **Set up a custom domain**
3. Enter your domain (e.g. `yourname.com`)
4. Add `www.yourname.com` as a second custom domain

Cloudflare issues and renews HTTPS certificates automatically.
<br>

### 9.4 Update baseURL

Edit `hugo.yaml` to match your domain:

```yaml
baseURL: "https://yourname.com/"
```

Then push the change:

```bash
git add hugo.yaml
git commit -m "Update baseURL to custom domain"
git push
```

Your site will be live at your custom domain within a minute or two.
<br>

---
<br>

## Phase 10: Day-to-Day Workflow

Once set up, publishing a new post takes four steps:

```bash
# 1. Create the post
hugo new blog/my-new-post.md

# 2. Write your content, then set draft: false

# 3. Preview locally
hugo server -D

# 4. Publish
git add .
git commit -m "New post: My new post title"
git push
```

Cloudflare Pages detects the push, rebuilds the site, and your post is live within seconds — no servers to manage, no deployment scripts, no fees.

---

## Troubleshooting

**Post not appearing on the live site**

Check that `draft: false` is set in the post's frontmatter. Drafts are skipped during production builds.

**Build fails on Cloudflare Pages**

The most common cause is a Hugo version mismatch. Go to your Pages project → **Settings** → **Environment variables** and confirm `HUGO_VERSION` matches your local version exactly (`hugo version`).

**Blank page or theme not loading**

Verify the theme line in `hugo.yaml` says `PaperMod` exactly (capital P, capital M). Also confirm that `themes/PaperMod` is not an empty folder — if it is, run `git submodule update --init --recursive`.

**CSS or formatting looks broken**

Make sure you have the **extended** version of Hugo installed. Run `hugo version` and look for the word "extended" in the output. If it is missing, reinstall using the `.deb` package from the releases page.

**Domain not resolving after nameserver change**

DNS propagation can take up to 48 hours. Check progress at [dnschecker.org](https://dnschecker.org) by entering your domain.

**Push rejected by GitHub**

If you see a "remote rejected" error and have two-factor authentication enabled, make sure you are using a Personal Access Token as your password, not your account password.
<br>

---
<br>

## Folder Structure Reference

```
my-blog/
├── archetypes/
│   └── default.md
├── content/
│   ├── blog/
│   │   ├── _index.md          ← Blog section index
│   │   └── my-first-post.md   ← Your first post
│   ├── about.md               ← About Me page
│   └── links.md               ← Links page
├── hugo.yaml                  ← Main config file
├── .gitignore
├── themes/
│   └── PaperMod/              ← Theme (submodule)
├── static/                    ← Images go here
├── layouts/                   ← Custom layout overrides
└── data/
```

To use an image in a post, place it in `static/images/` and reference it as:

```markdown
![Description](/images/photo.jpg)
```

---

That's everything. You now have a fully automated publishing pipeline: write a post, push to GitHub, and it's live within seconds — served globally over HTTPS at no cost.
