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
permissions:
  issues: write

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
          script: |
            const issueTitle = context.payload.issue.title;
            const issueNumber = context.payload.issue.number;
            const repo = context.repo.repo;
            const owner = context.repo.owner;

            console.log(`Checking for issues similar to: "${issueTitle}" (Issue #${issueNumber})`);

            // Fetch all open issues
            const { data: issues } = await github.rest.issues.listForRepo({
              owner,
              repo,
              state: "open",
              per_page: 50
            });

            console.log(`Found ${issues.length} total open issues`);

            // Common words to ignore
            const commonWords = new Set([
              'the', 'a', 'an', 'and', 'or', 'but', 'in', 'on', 'at', 'to', 'for',
              'is', 'are', 'was', 'were', 'will', 'be', 'has', 'have', 'had',
              'this', 'that', 'these', 'those', 'it', 'its', 'with', 'from'
            ]);

            // Split and clean the title words
            const titleWords = issueTitle
              .toLowerCase()
              .split(/\s+/)
              .filter(word => {
                // Remove common words and require words to be at least 3 characters
                return !commonWords.has(word) && word.length > 2;
              });

            console.log(`Keywords from new issue: ${titleWords.join(', ')}`);

            const similarIssues = issues.filter(issue => {
              if (issue.number === issueNumber) return false;
              
              const otherTitle = issue.title.toLowerCase();
              const otherWords = otherTitle.split(/\s+/);
              
              // Count how many words match
              const matchingWords = titleWords.filter(word => 
                otherWords.includes(word)
              );
              
              // Require at least 2 matching words or 1 if the title only has 1 significant word
              const minMatchesNeeded = titleWords.length === 1 ? 1 : 2;
              const isMatch = matchingWords.length >= minMatchesNeeded;
              
              if (isMatch) {
                console.log(`Match found: Issue #${issue.number} - "${issue.title}"`);
                console.log(`Matching words: ${matchingWords.join(', ')}`);
              }
              
              return isMatch;
            });

            console.log(`Found ${similarIssues.length} similar issues`);

            if (similarIssues.length > 0) {
              let commentBody = "🔍 **Similar Issues Found**\n\n";
              commentBody += "| Issue | Summary | Labels |\n";
              commentBody += "|-------|---------|--------|\n";
              
              similarIssues.forEach(issue => {
                const labels = issue.labels.map(label => `\`${label.name}\``).join(', ') || '-';
                commentBody += `| [#${issue.number}](${issue.html_url}) | ${issue.title} | ${labels} |\n`;
              });

              commentBody += "\n Please review these similar issues before proceeding!";

              console.log('Creating comment with similar issues');
              
              await github.rest.issues.createComment({
                owner,
                repo,
                issue_number: issueNumber,
                body: commentBody
              });
              
              console.log('Comment created successfully');
            } else {
              console.log('No similar issues found - no comment needed');
            }
```

## License

This project is licensed under the MIT License.
