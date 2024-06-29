---
layout: home
title: fine tuning llms with documentation Wiki
---

# Welcome to fine tuning llms with documentation Wiki!

This wiki contains information about fine tuning llms with documentation.

## Topics

{% for post in site.posts %}

- [{{ post.title }}]({{ post.url | relative_url }})
  {% endfor %}

## Wiki Generation Script

Here's the Python script used to generate this wiki:

````python
# Import necessary libraries
import os
import time
import yaml
import re
import ast
from github import Github
from github.GithubException import GithubException
from git import Repo
import requests
import openai

# Set up configuration variables
OPENAI_API_KEY = "YOUR_OPENAI_API_KEY"  # Replace with your OpenAI API key
GITHUB_TOKEN = "YOUR_GITHUB_TOKEN"  # Replace with your GitHub personal access token
GITHUB_USERNAME = "YOUR_GITHUB_USERNAME"  # Replace with your GitHub username
GITHUB_ORG = ""  # Optional: Your GitHub organization name (leave empty for personal account)
REPO_NAME = "your-repo-name"  # Replace with your desired repository name
MAIN_TOPIC = "Your Main Topic"  # Replace with your main wiki topic

# Set up OpenAI client
client = openai.OpenAI(api_key=OPENAI_API_KEY)

# Function to interact with GPT-4
def ask_gpt(prompt):
    try:
        response = client.chat.completions.create(
            model="gpt-4",
            messages=[
                {
                    "role": "system",
                    "content": f"You are a helpful assistant that generates wiki learning content about {MAIN_TOPIC}.",
                },
                {"role": "user", "content": prompt},
            ],
        )
        return response.choices[0].message.content.strip()
    except Exception as e:
        print(f"An error occurred: {e}")
        time.sleep(20)  # Wait for 20 seconds before retrying
        return ask_gpt(prompt)

# Function to generate wiki title and description
def generate_wiki_info(topic):
    prompt = f"""
    Given the main topic "{topic}", please provide:
    1. A concise wiki title (5 words or less)
    2. A brief wiki description (15 words or less)

    Format the response as a Python dictionary with keys 'title' and 'description'.
    """
    response = ask_gpt(prompt)
    try:
        info = ast.literal_eval(response)
        return info["title"], info["description"]
    except (SyntaxError, ValueError, KeyError) as e:
        print(f"Error parsing GPT response for wiki info: {e}")
        return f"{topic} Wiki", f"A wiki about {topic}"

# Generate wiki title and description
WIKI_TITLE, WIKI_DESCRIPTION = generate_wiki_info(MAIN_TOPIC)

print(f"Wiki Title: {WIKI_TITLE}")
print(f"Wiki Description: {WIKI_DESCRIPTION}")
print(f"Repository Name: {REPO_NAME}")

# Generate table of contents
toc = ask_gpt(
    f"Generate a list of 7 important topics for a wiki about {MAIN_TOPIC} for someone who wants to learn and understand {MAIN_TOPIC} and any of its sub-topics. Format the response as a Python list without any markdown formatting or code block indicators."
)

# Process the table of contents response
toc = toc.strip()
if toc.startswith("```") and toc.endswith("```"):
    toc = toc[3:-3]
if toc.startswith("python"):
    toc = toc[6:]

try:
    topics = ast.literal_eval(toc)
    if not isinstance(topics, list):
        raise ValueError("The result is not a list")
except (SyntaxError, ValueError) as e:
    print(f"Error parsing GPT response: {e}")
    print("GPT response:", toc)
    topics = []

topics = [str(topic) for topic in topics]

# Create necessary directories for Jekyll
os.makedirs("docs", exist_ok=True)
os.makedirs("docs/_posts", exist_ok=True)

# Function to generate wiki content for each topic
def generate_wiki_content(topic, main_topic):
    prompt = f"""Write a detailed wiki article about '{topic}' in the context of {main_topic}.
    Structure your response as follows:

    1. Start with a brief introduction (2-3 sentences).
    2. Provide a table of contents using Markdown list format.
    3. For each section in the table of contents:
       - Use a second-level heading (##) for the section title
       - Write detailed content for that section
       - Include relevant examples or use cases where appropriate
       - If applicable, mention any tools or technologies related to this topic
    4. End with a "Further Reading" section that suggests related topics or resources.

    Ensure all content is in proper Markdown format. Do not include a conclusion section.
    """
    return ask_gpt(prompt)

# Generate content for each topic
for topic in topics:
    print(f"Generating content for: {topic}")
    content = generate_wiki_content(topic, MAIN_TOPIC)

    front_matter = {"layout": "post", "title": topic, "categories": [MAIN_TOPIC]}
    safe_topic = re.sub(r"[^\w\-]", "-", topic.lower())
    filename = f"docs/_posts/{time.strftime('%Y-%m-%d')}-{safe_topic}.md"

    with open(filename, "w", encoding="utf-8") as f:
        f.write("---\n")
        yaml.dump(front_matter, f)
        f.write("---\n\n")
        f.write(content)

# Generate index page
index_content = f"""---
layout: home
title: {WIKI_TITLE}
---

# Welcome to {WIKI_TITLE}!

This wiki contains information about {MAIN_TOPIC}.

## Topics

{{% for post in site.posts %}}
- [{{{{ post.title }}}}]({{{{ post.url | relative_url }}}})
{{% endfor %}}
"""

with open("docs/index.md", "w", encoding="utf-8") as f:
    f.write(index_content)

# Generate _config.yml for GitHub Pages
config_content = f"""
title: {WIKI_TITLE}
description: {WIKI_DESCRIPTION}
theme: jekyll-theme-primer
baseurl: "/{REPO_NAME}"
url: "https://{GITHUB_USERNAME}.github.io"

permalink: /:year/:month/:day/:title
plugins:
  - jekyll-relative-links
  - jekyll-seo-tag

relative_links:
  enabled: true
  collections: true

markdown: kramdown
kramdown:
  input: GFM
  syntax_highlighter: rouge
"""

with open("docs/_config.yml", "w", encoding="utf-8") as f:
    f.write(config_content)

print("Wiki generation complete!")

# Create GitHub repository, push content, and enable GitHub Pages
g = Github(GITHUB_TOKEN)

# Function to create or get existing repository
def create_or_get_repo(g, repo_name, description):
    try:
        user = g.get_user()
        try:
            repo = user.create_repo(repo_name, description=description)
            print(f"Repository created: {repo.html_url}")
        except GithubException as e:
            if e.status == 422:
                print(f"Repository '{repo_name}' already exists. Attempting to access it.")
                repo = user.get_repo(repo_name)
                print(f"Using existing repository: {repo.html_url}")
            else:
                print(f"Error creating repository: {e}")
                return None
        return repo
    except Exception as e:
        print(f"Unexpected error: {e}")
        return None

repo = create_or_get_repo(g, REPO_NAME, WIKI_DESCRIPTION)

if repo is None:
    print("Failed to create or access the repository. Exiting.")
    sys.exit(1)

# Initialize git repo and push to GitHub
local_repo = Repo(".")
try:
    origin = local_repo.remote("origin")
    print("Remote 'origin' already exists.")
except ValueError:
    origin = local_repo.create_remote("origin", repo.clone_url)
    print(f"Created remote 'origin' with URL: {repo.clone_url}")

# Add all files
local_repo.git.add(A=True)

# Check current branch name
try:
    current_branch = local_repo.active_branch.name
except TypeError:
    current_branch = None

if current_branch is None:
    local_repo.index.commit("Initial commit with wiki content")
    local_repo.git.branch("main")
elif current_branch != "main":
    local_repo.git.branch("main", m=True)

# Ensure we're on the 'main' branch
local_repo.git.checkout("main")

# Commit all changes
local_repo.git.add(A=True)
local_repo.index.commit("Add wiki content, config, and index")

# Pull before pushing to avoid conflicts
try:
    origin.pull("main")
except Exception as e:
    print(f"Error pulling from remote: {e}")

# Push to the remote repository
try:
    origin.push("main")
    print(f"Content pushed to {repo.html_url}")
except Exception as e:
    print(f"Error pushing to remote: {e}")

# Enable GitHub Pages
try:
    url = f"https://api.github.com/repos/{repo.full_name}/pages"
    headers = {
        "Authorization": f"token {GITHUB_TOKEN}",
        "Accept": "application/vnd.github.v3+json",
    }
    data = {"source": {"branch": "main", "path": "docs"}}
    response = requests.post(url, json=data, headers=headers)
    if response.status_code == 201:
        print("GitHub Pages enabled successfully")
    elif response.status_code == 409:
        print("GitHub Pages is already enabled. Updating settings...")
        response = requests.put(url, json=data, headers=headers)
        if response.status_code == 200:
            print("GitHub Pages settings updated successfully")
        else:
            print(f"Failed to update GitHub Pages settings. Status code: {response.status_code}")
    else:
        print(f"Failed to enable GitHub Pages. Status code: {response.status_code}")
        print(f"Response: {response.text}")
except Exception as e:
    print(f"Error configuring GitHub Pages: {e}")

# Set the homepage URL
repo.edit(homepage=f"https://{GITHUB_USERNAME}.github.io/{REPO_NAME}/")

if GITHUB_ORG:
    print(f"Your wiki should be available at: https://{GITHUB_ORG}.github.io/{REPO_NAME}/")
else:
    print(f"Your wiki should be available at: https://{GITHUB_USERNAME}.github.io/{REPO_NAME}/")
print("It may take a few minutes for the site to be built and become accessible.")
```
````
