---
title: "Custom GPT for GitHub PR reviews"
date: 2023-11-17
draft: true
tags:
  - AI
  - ChatGPT
authors:
  - tomas-panik
resources:
  - name: create-a-gpt
    src: "images/create-a-gpt.png"
    title: Create a GPT button
  - name: title-and-description
    src: "images/title-and-description.png"
    title: Title and Description
  - name: gpt-authentication
    src: "images/gpt-authentication.png"
    title: GPT Authentication
  - name: pr-review-demo
    src: "images/pr-review-demo.gif"
    title: DEMO
  - name: pr-review-demo2
    src: "images/pr-review-demo2.gif"
    title: DEMO
---

OpenAI recently introduced [custom GPT's](https://openai.com/blog/introducing-gpts). Lets briefly sum-up what GPT's are compared to classical ChatGPT: 

1. It contains custom predefined instructions
2. It has custom conversation starters
3. It can now contain multiple capabilities at once, in classic ChatGPT you had to choose one. Those are:
    * Web browsing
    * DALL-E image generation
    * Code interpreter
    * Plugins - plugins got kind of deprecated, it is now called "custom Actions"

What is especially interesting is last part, where it is much easier now to create integrations with custom actions. Developers no longer need to create plugin (which was complex process), but it can be now done by anyone. 

So lets try to build GPT that will help developers with PR reviews. 

# Creating GPT
Starting with custom GPT you must have GPT Plus subscription. It can be created by clicking `Explore` and than `Create a GPT`

{{< img name="create-a-gpt" lazy=false size="origin" >}}

# Goal
Now lets review how we want to use GPT:

* I want to paste it GitHub PR Url which I want it to review
* It will download diff of PR using GitHub API (authenticate as me so it will have access even to private repositories)
* It will review diff, analyze it for bugs, code smells and suggest improvements
* It will write back those findings and it will ask me if I want to create PR review in pending state using those findings as comments
* By typing "Yes" it will submit PR review

So it will be assisting with PR reviews. I want to use GPT's comments just as a hint what I should look at, so I will be updating/deleting his comments most of the time. So we must make sure that review submitted to GitHub remains in PENDING state. And those comments must be recognizable so will prefix with `By GPT:`.

# GPT configuration
So lets go ahead and configure GPT, we will skip GPT Builder and configure everything manually.

## Title and description
I set following

Name: `GitHub PR Code Reviewer`
Description: `Expert at GitHub PR code reviews, using GitHub API for insightful feedback.`

{{< img name="title-and-description" lazy=false size="origin" >}}

## Actions

before configuring instructions, lets configure actions first, as it is fundamental part of the configuration.
Actions are manifest of [OpenAPI specification](https://spec.openapis.org/oas/v3.1.0) written in JSON format. 

Lets click `Create new actions` in GPT configuration and set following schema:
```json
{
  "openapi": "3.1.0",
  "info": {
    "title": "GitHub API",
    "description": "Retrieves the diff of a specified pull request from a GitHub repository as well as submitting PR review.",
    "version": "v1.0.0"
  },
  "servers": [
    {
      "url": "https://api.github.com"
    }
  ],
  "paths": {
    "/repos/{owner}/{repo}/pulls/{pull_number}/files": {
      "get": {
        "description": "Get the diff of individual files in PullRequest.",
        "operationId": "GetPullRequestDiff",
        "parameters": [
          {
            "name": "owner",
            "in": "path",
            "description": "Owner of the repository.",
            "required": true,
            "schema": {
              "type": "string"
            }
          },
          {
            "name": "repo",
            "in": "path",
            "description": "Repository name.",
            "required": true,
            "schema": {
              "type": "string"
            }
          },
          {
            "name": "pull_number",
            "in": "path",
            "description": "Pull request number.",
            "required": true,
            "schema": {
              "type": "integer"
            }
          }
        ],
        "responses": {
          "200": {
            "description": "Success",
            "content": {
              "application/json": {
                "schema": {
                  "type": "array",
                  "items": {
                    "$ref": "#/components/schemas/FileDiff"
                  }
                }
              }
            }
          },
          "404": {
            "description": "Not Found"
          }
        }
      }
    },
    "/repos/{owner}/{repo}/contents/{path}": {
      "get": {
        "description": "Get the content of a file in a repository.",
        "operationId": "GetFileContent",
        "parameters": [
          {
            "name": "owner",
            "in": "path",
            "description": "Owner of the repository.",
            "required": true,
            "schema": {
              "type": "string"
            }
          },
          {
            "name": "repo",
            "in": "path",
            "description": "Repository name.",
            "required": true,
            "schema": {
              "type": "string"
            }
          },
          {
            "name": "path",
            "in": "path",
            "description": "Path to the file.",
            "required": true,
            "schema": {
              "type": "string"
            }
          },
          {
            "name": "ref",
            "in": "query",
            "description": "The name of the commit/branch/tag. Default: the repository\u2019s default branch (usually master).",
            "required": false,
            "schema": {
              "type": "string"
            }
          }
        ],
        "responses": {
          "200": {
            "description": "Success",
            "content": {
              "application/json": {
                "schema": {
                  "$ref": "#/components/schemas/FileContent"
                }
              }
            }
          },
          "404": {
            "description": "Not Found"
          }
        }
      }
    },
    "/repos/{owner}/{repo}/pulls/{pull_number}/reviews": {
      "post": {
        "description": "Create a review for a pull request in a pending state.",
        "operationId": "SubmitPullRequestReview",
        "parameters": [
          {
            "name": "owner",
            "in": "path",
            "description": "Owner of the repository.",
            "required": true,
            "schema": {
              "type": "string"
            }
          },
          {
            "name": "repo",
            "in": "path",
            "description": "Repository name.",
            "required": true,
            "schema": {
              "type": "string"
            }
          },
          {
            "name": "pull_number",
            "in": "path",
            "description": "The number of the pull request.",
            "required": true,
            "schema": {
              "type": "integer"
            }
          }
        ],
        "requestBody": {
          "description": "Review details",
          "required": true,
          "content": {
            "application/json": {
              "schema": {
                "$ref": "#/components/schemas/PullRequestReview"
              }
            }
          }
        },
        "responses": {
          "200": {
            "description": "Review successfully submitted",
            "content": {
              "application/json": {
                "schema": {
                  "$ref": "#/components/schemas/ReviewResponse"
                }
              }
            }
          }
        }
      }
    }
  },
  "components": {
    "schemas": {
      "FileDiff": {
        "type": "object",
        "properties": {
          "sha": {
            "type": "string"
          },
          "filename": {
            "type": "string"
          },
          "status": {
            "type": "string"
          },
          "additions": {
            "type": "integer"
          },
          "deletions": {
            "type": "integer"
          },
          "changes": {
            "type": "integer"
          },
          "contents_url": {
            "type": "integer"
          },
          "patch": {
            "type": "string"
          }
        }
      },
      "FileContent": {
        "type": "object",
        "properties": {
          "type": {
            "type": "string"
          },
          "encoding": {
            "type": "string"
          },
          "size": {
            "type": "integer"
          },
          "name": {
            "type": "string"
          },
          "path": {
            "type": "string"
          },
          "content": {
            "type": "string"
          },
          "sha": {
            "type": "string"
          },
          "url": {
            "type": "string"
          },
          "git_url": {
            "type": "string"
          },
          "html_url": {
            "type": "string"
          },
          "download_url": {
            "type": "string"
          }
        }
      },
      "PullRequestReview": {
        "type": "object",
        "required": ["comments"],
        "properties": {
          "comments": {
            "type": "array",
            "description": "A list of review comments.",
            "items": {
              "$ref": "#/components/schemas/ReviewComment"
            }
          }
        }
      },
      "ReviewComment": {
        "type": "object",
        "properties": {
          "path": {
            "type": "string",
            "description": "The relative path to the file that the comment is attached to."
          },
          "position": {
            "type": "integer",
            "description": "The line index in the diff to place the comment."
          },
          "body": {
            "type": "string",
            "description": "Text of the comment."
          }
        }
      },
      "ReviewResponse": {
        "type": "object",
        "properties": {}
      }
    }
  }
}
```

We allow him to do 3 actions using GitHub API:
* GetPullRequestDiff (`GET` - `/repos/{owner}/{repo}/pulls/{pull_number}/files`) - download diff's for all files in PR
  * I tried to use `/repos/{owner}/{repo}/pulls/{pull_number}` with header `Accept` set to `application/vnd.github.v3.diff`, but it turns out that GPT is not allowed to set headers, so it can use only endpoints where overriding header is not necessary. 
* GetFileContent (`GET` - `/repos/{owner}/{repo}/contents/{path}`) - optional. I don't know if GPT will ever use this.
* SubmitPullRequestReview (`POST` - `/repos/{owner}/{repo}/pulls/{pull_number}/reviews`) - submitting PR review
  * Note that we don't define `event` property for body, because we don't want GPT to fill it. This will make sure that it will have always PR in pending state.

### Authentication
There are two ways how GPT can authenticate with API requests:
* API key - basic, bearer or custom
* OAuth

Although OAuth is probably better way I am going to use API key for now. Steps to generate it in GitHub are:
* Go to `https://github.com/settings/tokens`
* Click `Generate new token` (I use classic tokens)
* Fill in `Note`, `Expiration` by your preference and check in `Scopes` `repo` scope
* Click `Generate token` and store token for later use

Now, configure authentication in GPT configuration. 

{{< img name="gpt-authentication" lazy=false size="origin" >}}

## Instructions

Most important part are instructions. With a lot of fine tuning I came up with this:

```text
The primary role of 'GitHub PR Code Reviewer' is to assist in GitHub Pull Request code reviews. 

# When user request you to review PR by its URL
If the request is `Review PR` you will only answer `Which PR?` and wait for user to submit PR link. 
If the request is directly PR Url you will follow directly with review of such PR.

## Follow these steps in order
- Step 1: Download patches for individual changed files, analyze them and look for "Findings". 
- Step 2: If necessary query full file content for broader context
- Step 3: Respond your findings

## Explanation of "Findings"
- Finding can be code smell, issue or bug
- What you should look at in patches:
    - The code is well-designed.
    - The functionality is good for the users of the code.
    - Any UI changes are sensible and look good.
    - Any parallel programming is done safely.
    - The code isn't more complex than it needs to be.
    - The developer isn't implementing things they might need in the future but don't know they need now.
    - Code has appropriate unit tests.
    - Tests are well-designed.
    - The developer used clear names for everything.
    - Comments are clear and useful, and mostly explain why instead of what.
    - Code is appropriately documented.

## Explanation of "Position"
The `position` value equals the number of lines down from the first "@@" hunk header in the file you want to add a comment. The line just below the "@@" line is position 1, the next line is position 2, and so on. The position in the diff continues to increase through lines of whitespace and additional hunks until the beginning of a new file. Position is never range it is just single number.

for example:
* the line `uses: actions/checkout@v3` is position 5 in patch: `"@@ -36,7 +36,7 @@ jobs:\n       HUGO_VERSION: 0.111.3\n     steps:\n       - name: Checkout\n-        uses: actions/checkout@v3\n+        uses: actions/checkout@v4\n`
* the line `{{- if .Values.aws_fargate.enabled }}` is position 4 in patch: `@@ -1575,6 +1575,7 @@ receivers:\n    prometheus/node-metrics:\n      config:\n        scrape_configs:\n+{{- if .Values.aws_fargate.enabled }}\n          - job_name: 'kubernetes-nodes-cadvisor'`

## Style of the response
- In the response DO NOT iterate through files, but iterate through "Findings". There can be multiple "Findings" in single file, and there can be also none.
- For each "Finding" will always contain: 
  - Full path to the file
  - Line of code - you will infer this from diff
  - "Position"
  - Explanation of the finding and suggested improvement. 

Follow structure for Findings:
path: <Full path to the file>, lines: <line of code> , position: <"Position">
<finding and suggested improvement>

Individual findings are separated by new line. Full path to the file and "Position" in path are bold. 

Example:
code/main.py:50
This line is unclear what it means, it would be better if you call variable `sumOfLines`.

path: doc/exported_metrics.md, lines: 5-6, position: 1
The documentation for exported metrics is unclear, to make it more clear write this `...`

- Try to suggest fix for the problem if possible
- If there is "Finding" in the file do not respond anything for that particular file
- IMPORTANT: Do not explain what the code is doing, focus on explanation of the "Findings" and improvements suggestions.
- You will always ask if the user wants to submit those findings as PR review

# When user request you to submit findings as PR review
You will use individual findings from your previous response and submit them as PR review in PENDING state, each finding as individual comment.
Here are rules of submitted comments:
* Set `path` parameter from Finding's full file path
* The `position` equals to already explained "Position"
* Set `body` parameter with actual Finding text
* IMPORTANT: Always prefix each comment with following text `**By GPT:** `.
* Full path to the file and line of code of Finding will never be included in comment body.
```

Explanation of instructions:
* When user writes `Review PR` (Which I will use as conversation started) GPT is instructed to just answer `Which PR?` to minimize text output for most common scenario. User can also write PR Url directly, but that wouldn't sound like a chat right?
* GPT is instructed how he should proceed with PR Url and also that he can optionally download also full file content for broader context.
* GPT is clearly instructed what is `Findings` and what he should look at in code. I used [summary of Google engineering practices](https://github.com/google/eng-practices/blob/master/review/reviewer/looking-for.md#summary)
* GPT is clearly instructed how he should respond with `Findings`. 
  * I had to set `IMPORTANT` on instruction to not explain what the code is doing, because GPT was doing that all the time.
* GPT is instructed how he should submit review
  * GPT was tending to set `position` incorrectly, he was setting it as line of code. So I instructed him with full documentation and even with example of how `position` is calculated. 

## Conversation starters
Lets set just one `Review PR`, this is message what we explicitly instructed GPT on.

# Lets test it
Now lets test it on one PR done by dependabot in repo of this blogpost: https://github.com/gantrior/tomas-panik-blog/pull/5

{{< img name="pr-review-demo" lazy=false size="origin" >}}

Great! It works. 

# A little tuning..
Now lets try if GPT can skip responding comments to the user and submit comments directly, which would speed up the review a little bit. 

Lets add following to the end of instructions:
```
# When user request you to review PR with review submission
If the request is `Review PR with review submission` you will only answer `Which PR?` and wait for user to submit PR link. After user provides PR url, you will review PR with the rules above, but do not print anything to the user, but you will assume that user wants you to submit findings as PR review. So you will submit review right away
```

And add new conversation starter: `Review PR with review submission` 

{{< img name="pr-review-demo2" lazy=false size="origin" >}}

A little bit faster now.

# Conclusion
We have built GPT that could help us doing code reviews more efficiently. Custom GPT's shows us great potential, so lets see what we can build with it next. I will follow with other ideas with upcoming blog posts.










