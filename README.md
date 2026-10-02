name: Update GitHub Profile

on:
  schedule:
    - cron: "0 */6 * * *"

  workflow_dispatch:

  push:
    branches:
      - main

permissions:
  contents: write

jobs:
  update-profile:
    runs-on: ubuntu-latest

    steps:

      - name: Checkout profile
        uses: actions/checkout@v4

      - name: Generate dynamic project section
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |

          python <<'PY'

          import os
          import urllib.request
          import json
          import html

          username = "andrewsemtetteh"

          url = (
              f"https://api.github.com/users/{username}/repos"
              "?per_page=100&sort=updated"
          )

          request = urllib.request.Request(
              url,
              headers={
                  "Accept": "application/vnd.github+json",
                  "User-Agent": username
              }
          )

          with urllib.request.urlopen(request) as response:
              repos = json.loads(response.read())

          # Remove forks and archived repositories
          repos = [
              repo for repo in repos
              if not repo.get("fork", False)
              and not repo.get("archived", False)
          ]

          # Sort by stars first, then recent activity
          repos.sort(
              key=lambda repo: (
                  repo.get("stargazers_count", 0),
                  repo.get("pushed_at", "")
              ),
              reverse=True
          )

          # Display up to 6 projects
          repos = repos[:6]

          cards = []

          for repo in repos:

              name = html.escape(repo["name"])

              description = repo.get("description") or \
                  "An experiment from my digital studio."

              description = html.escape(description)

              language = repo.get("language") or "CODE"

              stars = repo.get("stargazers_count", 0)

              url = repo["html_url"]

              card = f"""
          <a href="{url}">
            <img
              src="https://github-readme-stats.vercel.app/api/pin/?username={username}&repo={repo['name']}&theme=transparent&hide_border=true"
              width="48%"
            />
          </a>
          """

              cards.append(card)

          project_html = """

          <div align="center">

          """ + "\n".join(cards) + """

          </div>

          """

          with open("README.md", "r", encoding="utf-8") as file:
              readme = file.read()

          start_marker = "<!-- PROJECTS:START -->"
          end_marker = "<!-- PROJECTS:END -->"

          start = readme.index(start_marker)
          end = readme.index(end_marker)

          new_readme = (
              readme[:start]
              + start_marker
              + "\n"
              + project_html
              + "\n"
              + readme[end:]
          )

          with open("README.md", "w", encoding="utf-8") as file:
              file.write(new_readme)

          PY

      - name: Commit updated profile
        run: |

          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"

          git add README.md

          git diff --cached --quiet || \
            git commit -m "chore: update dynamic profile"

          git push
