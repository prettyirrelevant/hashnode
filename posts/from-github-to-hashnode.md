---
title: From GitHub to Hashnode - Automating Blog Post Publication via Custom GitHub Actions
description: This project involved building a GitHub Action to automatically publish Markdown/HTML files from a GitHub repository to Hashnode blog posts.
draft: false
tags: ["github actions","hashnode","automation","typescript","apihackathon"]
coverImageUrl: "https://res.cloudinary.com/dybhjquqy/image/upload/f_auto,q_auto/i3fbcu7cuvfucooefkh3"
---

Publishing blog posts often feels disconnected from my code and writing workflow. I end up with content stranded across various platforms. This gave me an idea - what if blog posts could live alongside my code in markdown files and publish automatically on `git push`?

## Solution Overview

The solution syncs files(Markdown and HTML for now) from a GitHub repository to my Hashnode blog. The workflow is simple:

- Create a GitHub repository and specify a directory for blog post files in markdown or other formats. I chose `posts/` for my setup.

- Set up a GitHub action workflow that utilizes the GitHub action I built. It looks something like this:

```yaml
# .github/workflows/publish.yml
name: Publish Blog Posts
on:
  push:

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: prettyirrelevant/hashnode-posts-publisher@v0.1.1
        with:
          supported-formats: md
          posts-directory: posts
          access-token: ${{ secrets.HASHNODE_TOKEN }}
          publication-id: ${{ secrets.HASHNODE_PUBLICATION_ID }}
```

- Get the **access token** and **publication ID** from your Hashnode dashboard and add to the GitHub repository as secrets. These allow publishing access. If stuck, the [setup guide[](https://github.com/prettyirrelevant/hashnode-posts-publisher?tab=readme-ov-file#inputs) has step-by-step instructions.

- Write a sample post like `first-post.md` as below, push changes, and voila - it's published on your Hashnode blog!
```txt
---
title: My First Post
description: My First Ever Blog Post From GitHub repo
tags: ["github", "hashnode", "automation", "first post"]
---

# My First Post
This is the content of my first blog post!
```

Importantly, it only publishes posts that have been updated. So making changes to a file will update the post on Hashnode, but unchanged files are skipped to avoid duplication.

Common GitHub repository files like README.md, LICENSE.md, and CONTRIBUTING.md are also excluded from publishing by default since they serve a different purpose.

## How I Built It

I kicked things off by diving into GitHub Actions. I went with TypeScript (other options were Shell Scripts, Javascript, etc.) since I'm familiar with it and appreciate the type safety.

My first order of business was figuring out the bare minimum features I'd need for a usable MVP. I settled on Markdown and HTML support, frontmatter parsing (critical for pulling out metadata from Markdown), uploading/updating posts to Hashnode, and maaybe drafts. Turns out drafts were a bust - the Hashnode API had other plans. Probably a skill issue lol.

After defining the inputs and outputs in action.yml, it was time to tango with the Hashnode API. Being my first GraphQL API, it was quite the adventure. Thankfully the docs were helpful and I got uploading/updating posts working, albeit no drafts. Win some, lose some! Here's a snippet of the action.yml:
```yml
author: 'Isaac Adewumi'
name: 'hashnode-posts-publisher'
description: 'Publish posts to Hashnode by pushing Markdown files to a GitHub repository.'

# Define input parameters
inputs:
  access-token:
    description: 'The Hashnode API access token.'
    required: true

  posts-directory:
    description: 'The local directory containing the blog posts.'
    required: false
    default: 'posts'

  supported-formats:
    description: 'The allowed file formats for blog posts.'
    required: false
    default: 'md'

  publication-id:
    description: 'The ID of the publication to publish the posts to.'
    required: true

runs:
  using: node20
  main: dist/index.js
```

With the API down, I built a wrapper around it using Axios:

```typescript
import axios, { isAxiosError } from 'axios'

import { UploadPostSuccessResponse, UpdatePostSuccessResponse, Post } from '../schema'

export class HashnodeAPI {
  private baseUrl = 'https://gql.hashnode.com'
  private publicationId: string
  client: axios.AxiosInstance

  constructor(accessToken: string, publicationId: string) {
    this.client = axios.create({
      headers: {
        Authorization: accessToken
      },
      timeout: 5000
    })
    this.publicationId = publicationId
  }

  async updatePost(post: Post, postId: string): Promise<UpdatePostSuccessResponse> {
    const query = `
    mutation UpdatePost($input: UpdatePostInput!) {
        updatePost(input: $input) {
          post {
            id
            slug
            url
          }
        }
      }
    `
    // rest of code..
  }

  async uploadPost(post: Post): Promise<UploadPostSuccessResponse> {
    const query = `
      mutation PublishPost($input: PublishPostInput!) {
        publishPost(input: $input) {
          post {
            id
            slug
            url
          }
        }
      }
    `
    // rest of code..
  }
}
```

Tests were passing - so far so good!

Now to wrangle those repository files thanks to the nifty [@actions/glob](https://github.com/actions/toolkit/tree/main/packages/glob) package. For markdown files, I parsed frontmatter to extra metadata. HTML files were converted to Markdown using [Turndown](https://github.com/mixmark-io/turndown) and necessary metadata was extracted using Regex.

That's when things got messy. Posts were uploading happily, but I soon realized they were duplicating on Hashnode with each push. No bueno. Enter my new best friend - the **lockfile**! This handy class tracks uploaded posts, solving my duplication troubles.

```typescript
interface Lockfile {
  content: LockfileContent[]
  repositoryName: string
  repositoryId: string
  createdAt: string
  updatedAt: string
  id: string
}

interface LockfileContent {
  hash: string
  path: string
  url: string
  id: string
}

export class LockfileAPI {
  constructor(repositoryId: string) {
    this.repositoryId = repositoryId
  }

  async updateLockfile({
    successfulUploads,
    currentLockfile
  }: {
    successfulUploads: PostSuccessResponse[]
    currentLockfile?: Lockfile
  }): Promise<UpdateLockfileResponse> {
    if (!currentLockfile) {
      currentLockfile = {
        id: '',
        repositoryName: process.env.GITHUB_REPOSITORY as string,
        createdAt: new Date().toISOString(),
        updatedAt: new Date().toISOString(),
        repositoryId: this.repositoryId,
        content: []
      }
    }

    // rest of code for handling various cases..
  }

  async retrieveLockfile(): Promise<RetrieveLockfileResponse> {
    // rest of code..
  }
}
```

Storing the lockfile was the next roadblock. GitHub cache expires after seven(7) days of inactivity, artifacts get complicated as I have to keep track of workflow identifiers...then it hit me - why not a simple Golang server! So I spun one up real quick ([check it out here](https://github.com/prettyirrelevant/hashnode-lockfile-server)). Not the most elegant solution, but it did the trick for now.

I even played around with integrating audio recordings at one point using Whisper and AI text generation. Let's just say that got messy real fast(translating a 10 minutes recording took quite the time) - maybe someday I'll revisit it!

The rest was spent testing, documenting, and writing this very post you're reading! It's been exhilarating to say the least, but now I've got a solid foundation to build and improve on.

## Next Steps

There's still room to improve and extend the project. Next, I could work on:

- Having to use an external server to manage a simple lockfile is annoying. It'd be much cleaner to just manage it within the repository.
- Supporting more formats like rST, AsciiDoc and LaTeX would be awesome. The more options for users, the better.

## Conclusion
Building an automated Hashnode publishing workflow for their API Hackathon was a total blast! I finally got around to diving into GitHub Actions - something I'd been meaning to learn.

The best part? I used this very workflow to author this [post](https://github.com/prettyirrelevant/hashode) in Markdown and push it straight from GitHub to publish. No more context switching between writing and publishing!

Mad props to the Hashnode team for bringing builders together to create solutions like this.

If you find this useful and use it for your own blog, I'd love to hear about it! Feel free to reach out to me on [X (formerly Twitter)](https://twitter.com/prettirrelevant)
```
