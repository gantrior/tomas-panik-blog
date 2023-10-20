---
title: "ChatGPT API Basics for Developers (in Python)"
date: 2023-10-19
draft: true
tags:
  - AI
  - Python
  - ChatGPT
authors:
  - tomas-panik
---

This article is for developers who haven't yet tried the ChatGPT API. I recently gave it a try and must conclude that it offers great potential. So, here's a detailed guide on how to get started.

{{< toc >}}

---

Before we start ..

## What is a "Token" and How Much Do OpenAI API Queries Cost?

In the context of OpenAI models, a token is a unit of text that the model processes. It can be a character, word, or compound word, depending on the language and specific text.

Token encoding is the process of breaking text down into individual tokens that the model can process. OpenAI uses the Byte-Pair-Encoding (BPE) algorithm for text tokenization.

For working with OpenAI tokens, you will need the `tiktoken` library.

```bash
pip install tiktoken
```

Use the `tiktoken` library to encode text into tokens.

```python
import tiktoken

tokenizer = tiktoken.encoding_for_model("gpt-3.5-turbo")
tokens = tokenizer.encode("Text that needs to be tokenized: !@#$%^&*()")

print(tokens)

for token in tokens:
  print(tokenizer.decode_single_token_bytes(token))
```
Output:
```bash
[1199, 430, 3966, 311, 387, 4037, 1534, 25, 758, 31, 49177, 46999, 5, 9, 368]
b'Text'
b' that'
b' needs'
b' to'
b' be'
b' token'
b'ized'
b':'
b' !'
b'@'
b'#$'
b'%^'
b'&'
b'*'
b'()'
```

From the output, it's clear how text is converted into tokens. For example, the word "Text" is 1 token. Conversely, the word "tokenized" is divided into 2 tokens: " token", "ized". Special characters are usually tokenized as 1 token per character. English language can be usually tokenized with less tokens than other languages due to lack of special characters.

### How Much Do OpenAI API Queries Cost?
API queries to OpenAI are not free, but new users have an initial budget sufficient for initial experimentation. The price generally depends on the input/output tokens.

The `gpt-3.5-turbo` model, which we will use, is very cost-effective. Its price is $0.0015/1000 input tokens and $0.002/1000 output tokens.

The complete price list is available here: [OpenAI Pricing](https://openai.com/pricing)

## Setting up the Environment for Working with the OpenAI GPT-3.5 API

### Installing Necessary Libraries
To interact with the OpenAI API, we will need the `openai` library. Install it using the following command:

```bash
pip install openai
```

### Creating an API Key for Authenticating Our Requests to the OpenAI API
The procedure is as follows:

* Go to https://platform.openai.com/account/api-keys.
* If you don't have an OpenAI account, create one.
* After logging in, click the button to create a new API key.
* Enter a name for your key and click "Create new secret key".

### Setting Up OpenAI API Credentials
To set up your API key, enter the following code:

```python
import openai

# Replace 'sk-abcdefgh1234567890' with your API key
openai.api_key = 'sk-abcdefgh1234567890'
```

Our environment is now prepared so we can start sending our first requests to ChatGPT via the OpenAI GPT-3.5 API!

## Sending a Basic Request
We will send the request using `openai.ChatCompletion.create` as follows:

```python
response = openai.ChatCompletion.create(
    model="gpt-3.5-turbo",
    messages=[
        {"role": "system", "content": "You are my assistant."},
        {"role": "user", "content": "Translate the following text to French: 'Hello, how are you?'"},
    ]
)
print(response['choices'][0]['message']['content'].strip())
```

```bash
Bonjour, comment ça va ?
```

## Understanding the Response
The API request returns a JSON-formatted response containing several useful pieces of information. However, we are only interested in the `choices` key, which contains an array of answers. In our case, there is only one answer containing a text string, which is the model's output. We use the `strip()` method to remove any extra white spaces from the output.

## Explanation of "Roles"
Roles tell the model how it should process the content. Here is the description of roles:

* **user** - informs the model that the subsequent content comes from the user – i.e., the person who asked the question,
* **assistant** - informs the model that the content was generated in response to the user,
* **system** - a message specified by the developer to "guide" the model's answer. Depending on the model used, the system message can have more or less impact on the actual model answers.

## Working with Conversation Context
When working with ChatGPT, it is essential to understand how the API handles the context of the conversation. If you want the API to consider previous responses in the context, you need to send the entire conversation history. This way, the model can see the entire conversation and respond consistently with previous messages.

```python
conversation_history = [
        {"role": "system", "content": "You are my assistant."},
        {"role": "user", "content": "Translate the following text to French: 'Hello, how are you?'"},
        {"role": "assistant", "content": "Bonjour, comment ça va ?"},
        {"role": "user", "content": "Translate the same to Spanish"},
    ]

response = openai.ChatCompletion.create(
    model="gpt-3.5-turbo",
    messages=conversation_history
)

print(response['choices'][0]['message']['content'].strip())
```

```bash
Hola, ¿cómo estás?
```

In this example, we sent four messages as part of the API request: one system message, two user messages, and one assistant message. The last user message is a new question we want the assistant to answer. Since we sent the entire conversation history, the assistant can see previous questions and answers and thus better respond to the new question.

## Some Advanced API Features

### Maximum Response Length
Limit the response length by setting the `max_tokens` parameter. This is useful for controlling the length of the output text.

```python
response = openai.ChatCompletion.create(
    model="gpt-3.5-turbo",
    messages=[
        {"role": "system", "content": "You are my assistant."},
        {"role": "user", "content": "Write a short bedtime story for my son (maximum 50 words)."},
    ],
    max_tokens=50
)
print(response['choices'][0]['message']['content'].strip())
```

```text
Once upon a time, in a faraway land, there was a curious little bunny named Benny. Every night, Benny would explore the magical forest, meeting talking animals and having amazing adventures. But when the moon smiled down, Benny knew it was time
```

### "Temperature"
The `temperature` parameter influences the creativity of the answer. A smaller value (e.g., 0.2) generates more consistent, less creative text, while a bigger value (e.g., 0.8) generates more creative but less predictable text.

```python
response = openai.ChatCompletion.create(
    model="gpt-3.5-turbo",
    messages=[
        {"role": "system", "content": "You are my assistant."},
        {"role": "user", "content": "Write a short bedtime story for my son (maximum 50 words)."},
    ],
    temperature=1
)
print(response['choices'][0]['message']['content'].strip())
```

```text
Once upon a time, in a magical forest, a little bear named Benny dreamed big. With determination and kindness, he made friends with all the animals. Together, they learned that no dream is too big when you have love and friendship. Goodnight, sweet dreams, my little adventurer!
```

### Additional Parameters
You can explore additional parameters in the documentation: [OpenAI API Documentation](https://platform.openai.com/docs/api-reference/chat/create)

## Limiting Tokens in Queries

Since OpenAI API calls are paid for tokens and the models have a predefined number of tokens they can work with at once, it's important to know how to limit them in API queries. When the number of tokens exceeds the limit, the API will return an error and won't execute.

There are multiple ways to handle larger messages; they can be broken down into smaller messages. I trimmed the excess parts:

```python
import tiktoken

def send_chatcompletion(
    system_prompt="You are my assistant",
    user_prompt=None,
    chat_model="gpt-3.5-turbo",
    completions_token_limit=500,
):
    response = openai.ChatCompletion.create(
        model=chat_model,
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt},
        ],
        temperature=1,
        max_tokens=completions_token_limit
    )

    return response

def send(
    system_prompt="You are my assistant",
    user_prompt=None,
    chat_model="gpt-3.5-turbo",
    model_token_limit=4097,
    completions_token_limit=500,
):
    tokenizer = tiktoken.encoding_for_model(chat_model)
    token_integers_system = tokenizer.encode(system_prompt)
    token_integers_user = tokenizer.encode(user_prompt)

    wrapped_content_token_count = 11  # Magic value that expresses the constant overhead for a request. It is derived from the number of messages and parameters sent in every request.
    user_token_limit = model_token_limit - len(token_integers_system) - wrapped_content_token_count - completions_token_limit

    user_prompt_trimmed = tokenizer.decode(token_integers_user[0:user_token_limit])

    return send_chatcompletion(
        system_prompt=system_prompt,
        user_prompt=user_prompt_trimmed,
        chat_model=chat_model,
        completions_token_limit=completions_token_limit,
    )

very_long_query = "Very long query" * 1600

tokenizer = tiktoken.encoding_for_model("gpt-3.5-turbo")
tokens = tokenizer.encode(very_long_query)
print(len(tokens))

send(user_prompt=very_long_query)
```

```text
4800
<OpenAIObject chat.completion id=chatcmpl-8AhQeDi7oBFwudJIVgwyl3KJor0pG at 0x7afa56becd60> JSON: {
  "id": "chatcmpl-8AhQeDi7oBFwudJIVgwyl3KJor0pG",
  "object": "chat.completion",
  "created": 1697560912,
  "model": "gpt-3.5-turbo-0613",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "I apologize, but I'm unable to assist with such a long query. Could you please provide a shorter, more specific query?"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 3597,
    "completion_tokens": 26,
    "total_tokens": 3623
  }
}
```

## Sample Project: Automated RSS Reader
In this project, we will create an automated RSS reader that allows us to follow topics of interest using personalized filters. This reader will load articles from an RSS feed, analyze them, and recommend articles to us based on our interests.

### Preparation
First, we'll need the `feedparser` library for loading RSS feeds and `tiktoken` for working with tokens.

```bash
pip install feedparser tiktoken
```

### Loading the RSS Feed
We will use the `feedparser` library to load the RSS feed from a selected source.

```python
import feedparser

rss_url = 'https://dev.to/feed/'
feed = feedparser.parse(rss_url)

print(feed)
```

### Creating a Custom Profile
A custom profile can be created in any comprehensible format.

```python
profile = """
Content: I am interested in one of the following topics:
* artificial intelligence
* programming in one of the languages Python, Golang, C# or frontend development in Typescript - React and Angular
* hardware technological innovations
* software updates, or news in new versions of existing software
""".strip()
```

### Article Analysis
And now, the final code for article analysis:

```python
import tiktoken
import json

# Function to send chat completion request to a chat model
def send(
    system_prompt=None,
    user_prompt=None,
    chat_model="gpt-3.5-turbo",
    model_token_limit=4097,
):
    tokenizer = tiktoken.encoding_for_model(chat_model)
    token_integers_system = tokenizer.encode(system_prompt)
    token_integers_user = tokenizer.encode(user_prompt)

    completions_token_limit = 500
    wrapped_content_token_count = 11 # Magic value representing the constant overhead for each request. Derived from the number of messages and parameters sent in each request.
    user_token_limit = model_token_limit - len(token_integers_system) - wrapped_content_token_count - completions_token_limit

    user_prompt_trimmed = tokenizer.decode(token_integers_user[0:user_token_limit])

    response = openai.ChatCompletion.create(
        model=chat_model,
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt_trimmed},
        ],
        temperature=1,
        max_tokens=completions_token_limit
    )

    return response

# Function to check if an article is relevant based on user's profile
def is_article_relevant(article_title, article_description, profile):
    prompt = """
Give me my profile and an article, and you'll evaluate if the article will interest me. You will answer in JSON format with the following structure: `{{ "answer": "<yes or no>", "reason": "<short explanation why it should or shouldn't interest me>"}}`.
My profile:
'''
{profile}
'''

Article title: `{title}`
""".format(profile=profile, title=article_title, description=article_description)

    response = send(
        system_prompt="You are my automated RSS article reader, evaluating whether an article is interesting for me.",
        user_prompt=prompt,
    )

    result = json.loads(response['choices'][0]['message']['content'].strip().lower())
    print(result)

    if result["answer"] == 'yes':
      return True

    return False

# Go through the first 10 articles in the feed
for entry in feed.entries[0:10]:
    article_title = entry.title
    article_description = entry.description
    if is_article_relevant(article_title, article_description, profile):
        print(f"RECOMMENDED ARTICLE: {entry.link}")
    else:
        print(f"Not recommended article: {article_title}")
```

Where the output is as follows:

```text
{'answer': 'no', 'reason': 'the article is not relevant to your interests. it focuses on javascript for beginners, while your interests lie in programming languages python, golang, c#, frontend development in typescript - react and angular.'}
Not recommended article: Website's best for JavaScript beginners!
{'answer': 'no', 'reason': 'the article is not related to any of the topics you mentioned in your profile.'}
Not recommended article: Amazon Rolls Out Passkeys
{'answer': 'no', 'reason': "the article is about cron jobs in next.js, which is a framework for frontend development with react. it doesn't cover any of the topics you mentioned in your profile."}
Not recommended article: Automate repetitive tasks with Next.js cron jobs
{'answer': 'no', 'reason': 'the article is about sending notifications in web apps, which does not align with your interests in artificial intelligence, programming, hardware innovations, or software updates.'}
Not recommended article: Sending Notifications In Your Web Apps
{'answer': 'no', 'reason': 'the article is about best practices for code reviews and pull requests, which may not directly align with your interests in artificial intelligence, programming languages, hardware innovations, or software updates.'}
Not recommended article: 13 tips for better Pull Requests and Code Review
{'answer': 'no', 'reason': 'the article is about a specific tool (zodot) for data validation in godot, which does not align with your interests in artificial intelligence, programming languages, hardware innovations, or software updates. it seems unrelated to your profile.'}
Not recommended article: Godot Data Validation with Zodot
{'answer': 'no', 'reason': "the article focuses on open-source projects for developers, but it doesn't specifically mention any topics from your profile such as artificial intelligence, programming languages, hardware innovations, or software updates."}
Not recommended article: 24 Open-Source Projects for Developers in 2023 🔥👍
{'answer': 'yes', 'reason': 'the article is about frontend development using react and typescript, which matches your interest.'}
RECOMMENDED ARTICLE: https://dev.to/chaoocharles/build-and-deploy-a-full-stack-e-commerce-nextjs-13-reactjs-typescript-tailwind-prisma-stripe-45f2
{'answer': 'no', 'reason': 'the article is about off-page seo and backlink building, which does not align with your interests in artificial intelligence, programming, hardware innovations, or software updates.'}
Not recommended article: Off-Page SEO: How to Build Backlinks and Improve Your Domain Authority
{'answer': 'no', 'reason': 'the article does not seem to align with your interests. it is likely about the flank stack, which is not mentioned in your profile.'}
Not recommended article: FLaNK Stack Weekly 16 October 2023
```

ChatGPT recommended one article for me to read based on my profile.

## Conclusion
The OpenAI API for automating personal workflows has countless applications and has the potential to save a significant amount of time.
