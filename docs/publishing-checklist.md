# Publishing checklist

How to publish a post on redgeoff.com and cross-post it. Copy the checklist into the
post's PR description (or a GitHub issue) and tick items off as you go.

Replace `<slug>` with the post's file name, e.g. `ai-agents-test-gate`. The post lives
at `https://redgeoff.com/posts/<slug>/`, which is the canonical URL every cross-post
points back to.

| When | What |
| ---- | ---- |
| Before merging | Final checks and the post's `date` |
| Day 0, any time | Publish on redgeoff.com, submit to HackerNoon, prep the dev.to draft |
| Day 1, 6–8 AM PT | Hacker News, then publish on dev.to |
| Day 2, 8–9 AM PT | LinkedIn and X |
| When HackerNoon publishes | Check its canonical link, add it to Publications |
| A week later | Check Brevo signups |

Posting times are rules of thumb. What matters most is being free to reply to comments
for a few hours after each post goes up. Medium is skipped on purpose.

## Before merging the post PR

- [ ] Front matter `images:` points at the hero image. It becomes the link-preview card
      on LinkedIn, X and HackerNoon, so use something readable at thumbnail size (a
      wide diagram isn't). Credit it under the image, e.g. "Image credit: AI-generated".
- [ ] Front matter `description` is 160 characters or less, so it can double as
      HackerNoon's meta description.
- [ ] If the title is over 80 characters, pick a shorter version for Hacker News,
      ideally the first clause of the real title.
- [ ] Re-check any numbers readers will do arithmetic on: costs, rates, counts,
      multipliers.
- [ ] Preview with `hugo server`: hero image, figures, and the signup form at the bottom.
- [ ] Set `date` to a few minutes before you merge. Never set it in the future: Hugo
      skips future-dated posts, so the deploy quietly leaves it out. A stale date is a
      problem too, because Brevo's RSS campaign may skip items dated before its last
      check.

## Day 0: publish on redgeoff.com (evening is fine)

- [ ] Merge the PR. `pages.yml` deploys in under a minute; GitHub Pages can serve a
      cached 404 or old CSS for another minute or so.
- [ ] Open `https://redgeoff.com/posts/<slug>/` and check the hero image, figures and
      signup form.
- [ ] Check that `https://redgeoff.com/posts/index.xml` lists the post first, with the
      right `pubDate`.
- [ ] Paste the post URL into [LinkedIn Post Inspector](https://www.linkedin.com/post-inspector/).
      You should see the hero image, title, canonical URL and publish date. "No author
      found" is harmless.

### Brevo email (arrives within a day)

- [ ] A draft appears under **Marketing → Campaigns** (or the email sends, if the RSS
      campaign is set to send automatically). Review it and send.
- [ ] In Gmail, **Show original** on the email: From is `posts@redgeoff.com`, and SPF,
      DKIM and DMARC all say PASS.

### HackerNoon (submit on day 0; review takes a few days)

- [ ] Create a new story and paste in the post. Delete the list of site tag links at the
      bottom (`redgeoff.com/tags/...`) that comes along with the paste.
- [ ] "Is this story original on HackerNoon?" **No**, and enter
      `https://redgeoff.com/posts/<slug>/` with the trailing slash. This is the
      **First Seen At** canonical link; without it, editors treat the story as
      plagiarized.
- [ ] Backlink and distribution preference: **Backlink**. Max Readership drops the
      canonical link, so HackerNoon's copy outranks the original in search.
- [ ] Meta description: the front matter `description`, trimmed to 160 characters if
      needed.
- [ ] Turn on **AI-Assisted** if AI generated any images or text or helped with editing,
      and say how in the intro. HackerNoon checks submissions with GPTZero.
- [ ] Caption every image in HackerNoon's format: "(Source: AI-generated)",
      "(Source: author)", etc.
- [ ] Turn on **Vested Interest** if you hold a stock or product the post features.
- [ ] Up to 8 tags.
- [ ] End with: "Originally published on redgeoff.com, where you can subscribe for new
      posts by email."

### dev.to (prep on day 0, publish on day 1)

- [ ] The RSS import is already set up, so the post shows up as a draft in the dashboard
      within a few minutes of deploying. If it doesn't, check **Settings → Extensions**.
- [ ] In the draft's front matter: `canonical_url` is `https://redgeoff.com/posts/<slug>/`,
      up to 4 lowercase `tags`, and `published: false` for now.
- [ ] The hero image is already the first thing in the body. Either leave it, or set
      `cover_image` and delete the hero and its credit from the body. Not both, or it
      shows twice.

## Day 1 morning (6–8 AM PT): Hacker News

- [ ] Submit `https://redgeoff.com/posts/<slug>/` with a title of 80 characters or
      less. Leave the text box empty. Use a regular submission; Show HN is for things
      people can try.
- [ ] Straight away, add a first comment starting "Author here": what the project is,
      what the post covers, and that you're not selling anything.
- [ ] Keep the next three or four hours free to reply. Don't ask anyone to upvote it;
      Hacker News penalizes voting rings.
- [ ] Publish the dev.to draft: set `published: true` and save.

## Day 2 morning (8–9 AM PT): LinkedIn and X

- [ ] LinkedIn post linking to redgeoff.com, not a cross-post, so readers land next to
      the signup form. Write it in your own voice: tell one story rather than a
      numbered list, and skip dramatic one-liners and hashtag strings. For a trading
      post, keep the "my own accounts, not investment advice" line.
- [ ] Tag companies whose products the post features (e.g. Alpaca) in a comment on your
      post, worded so it's clear any bug was yours, not theirs.
- [ ] Be around for the first hour or two to reply to comments.
- [ ] X post: 280 characters max, and a link always counts as 23.
- [ ] Optional: Reddit (r/ExperiencedDevs, r/programming). Read each subreddit's
      self-promotion rules first; many remove personal blog links.

## When HackerNoon publishes

- [ ] Check the live story's canonical link: view source and search for
      `rel="canonical"`. It should point to redgeoff.com, and there should be an "Also
      published here" link at the bottom. For the first cross-posted story (Sep 2026)
      HackerNoon applied neither, even with First Seen At set. If it's missing, contact
      HackerNoon support and ask them to apply the canonical link to
      `https://redgeoff.com/posts/<slug>/`.
- [ ] Open a PR adding it to the top of `content/publications.md`, using HackerNoon's
      title (they often retitle) and publish date.
- [ ] Optional: a one-line comment on your LinkedIn post saying HackerNoon picked it up.

## A week later

- [ ] Check signups in Brevo: **CRM → Lists → redgeoff.com blog subscribers #2**.

## One-time setup (already done)

For reference if something breaks:

- **Brevo** (free plan, 300 emails/day): the "redgeoff.com blog posts" RSS campaign
  reads `https://redgeoff.com/posts/index.xml` daily and emails new posts to the
  subscribers list. Sender `posts@redgeoff.com`, reply-to `redgeoff@gmail.com`. The
  domain is authenticated with records in Route 53: the `brevo-code` TXT value on
  `redgeoff.com` (alongside any other TXT values, not replacing them), two DKIM CNAMEs
  and a DMARC record.
- **Signup form:** a Brevo form in an iframe. Its URL is `newsletterFormURL` in
  `config.toml`; it renders at the end of every post and on `/subscribe/` via the
  `newsletter` partial and shortcode. reCAPTCHA v3 allows `sibforms.com` and
  `redgeoff.com`. If Brevo's form changes height, re-measure and adjust
  `.newsletter iframe` in `static/css/blog.css`.
- **Feed:** `layouts/_default/rss.xml` emits full content with absolute URLs, for the
  dev.to import and the Brevo email. Only pages under `content/posts/` are in the posts
  feed, so other pages (like `/subscribe/`) never trigger an email.
- **dev.to:** RSS import from the posts feed, with "Mark the RSS source as canonical
  URL" on.
- **HackerNoon profile:** the default call to action should be "Get new posts by email"
  → `https://redgeoff.com/subscribe/`, with "About me" → `https://redgeoff.com/about/`
  second.
