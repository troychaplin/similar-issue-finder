# Similar Issue Finder

This repository contains a GitHub Action that checks for similar issues when a new issue is opened. The action helps to reduce duplicate issues by suggesting existing issues that might be related.

## How It Works

When a new issue is opened, the action will:

1. Fetch all open issues in the repository.
2. Compare the title of the new issue with the titles of existing issues.
3. If similar issues are found, it will comment on the new issue with links to the similar issues.

## Usage

To use this action, ensure you have the following workflow file in your repository:

```yml
name: Check Similar Issues
on:
  issues:
    types: [opened]

jobs:
  check-similar:
    runs-on: ubuntu-latest
    steps:
      - name: Check for similar issues
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const issueTitle = context.payload.issue.title;
            const issueNumber = context.payload.issue.number;
            const repo = context.repo.repo;
            const owner = context.repo.owner;

            // Fetch all open issues
            const { data: issues } = await github.rest.issues.listForRepo({
              owner,
              repo,
              state: "open",
              per_page: 50
            });

            const similarIssues = issues.filter(issue =>
              issue.number !== issueNumber &&
              issue.title.toLowerCase().includes(issueTitle.toLowerCase().split(" ")[0]) // Basic similarity check
            );

            if (similarIssues.length > 0) {
              let commentBody = "🔍 **Similar issues found:**\n";
              similarIssues.forEach(issue => {
                commentBody += `- [#${issue.number}](${issue.html_url}) - ${issue.title}\n`;
              });

              commentBody += "\nPlease check these before proceeding! 🚀";

              await github.rest.issues.createComment({
                owner,
                repo,
                issue_number: issueNumber,
                body: commentBody
              });
            }
```

## License

This project is licensed under the MIT License.
