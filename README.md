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
name: Check Similar Issues v1

permissions:
  issues: write

on:
  issues:
    types: [opened, labeled]

jobs:
  check-similar:
    runs-on: ubuntu-latest
    steps:
      - name: Check for similar issues v1
        if: github.event.action == 'opened' || (github.event.action == 'labeled' && github.event.label.name == 'check for related')
        uses: actions/github-script@v7
        with:
          script: |
            // Simple function to get word stem (handles basic plurals and common endings)
            function getWordStem(word) {
              word = word.toLowerCase();
              // Handle plurals and common endings
              if (word.endsWith('ies')) return word.slice(0, -3) + 'y';
              if (word.endsWith('es')) return word.slice(0, -2);
              if (word.endsWith('s')) return word.slice(0, -1);
              if (word.endsWith('ing')) return word.slice(0, -3);
              if (word.endsWith('ed')) return word.slice(0, -2);
              return word;
            }

            // Function to check if a label is a version label
            function isVersionLabel(label) {
              // Match patterns like: 6.7, 5.6, etc.
              return /^\d+\.\d+$/.test(label.name);
            }

            // Function to create a table with issues
            function createIssueTable(issues, versionTableOnly = false) {
              if (issues.length === 0) return '';
              
              let table = "| Issue | Summary | Labels |\n";
              table += "|-------|---------|--------|\n";
              
              issues.forEach(issue => {
                const labels = versionTableOnly 
                  ? issue.labels
                      .filter(label => isVersionLabel(label))
                      .map(label => `\`${label.name}\``)
                      .join(', ') || '-'
                  : issue.labels
                      .map(label => `\`${label.name}\``)
                      .join(', ') || '-';
                
                table += `| [#${issue.number}](${issue.html_url}) | ${issue.title} | ${labels} |\n`;
              });
              
              return table;
            }

            const issueTitle = context.payload.issue.title;
            const issueNumber = context.payload.issue.number;
            const repo = context.repo.repo;
            const owner = context.repo.owner;

            console.log(`Checking for issues similar to: "${issueTitle}" (Issue #${issueNumber})`);

            // Fetch all open issues with pagination
            let issues = [];
            let page = 1;
            let fetchedIssues;

            do {
              const { data } = await github.rest.issues.listForRepo({
                owner,
                repo,
                state: "open",
                per_page: 5,
                page: page
              });
              fetchedIssues = data;
              issues = issues.concat(fetchedIssues);
              console.log(`Fetched ${fetchedIssues.length} issues from page ${page}`);
              page++;
            } while (fetchedIssues.length === 5);

            console.log(`Found ${issues.length} total open issues`);

            // Common words to ignore
            const commonWords = new Set([
              'the', 'a', 'an', 'and', 'or', 'but', 'in', 'on', 'at', 'to', 'for',
              'is', 'are', 'was', 'were', 'will', 'be', 'has', 'have', 'had',
              'this', 'that', 'these', 'those', 'it', 'its', 'with', 'from',
              'not', 'what', 'where', 'when', 'why', 'how', 'all', 'any', 'both',
              'each', 'few', 'more', 'most', 'other', 'some', 'such', 'than'
            ]);

            // Split and clean the title words, then get their stems
            const titleWords = issueTitle
              .toLowerCase()
              .split(/\s+/)
              .filter(word => {
                // Remove common words and require words to be at least 3 characters
                return !commonWords.has(word) && word.length > 2;
              })
              .map(word => getWordStem(word));

            console.log(`Keywords from new issue (stemmed): ${titleWords.join(', ')}`);

            const similarIssues = issues.filter(issue => {
              if (issue.number === issueNumber) return false;

              const otherTitle = issue.title.toLowerCase();
              const otherWords = otherTitle
                .split(/\s+/)
                .map(word => getWordStem(word));

              // Count how many stemmed words match
              const matchingWords = titleWords.filter(word =>
                otherWords.includes(word)
              );

              // Require at least 2 matching words or 1 if the title only has 1 significant word
              const minMatchesNeeded = titleWords.length === 1 ? 1 : 2;
              const isMatch = matchingWords.length >= minMatchesNeeded;

              if (isMatch) {
                console.log(`Match found: Issue #${issue.number} - "${issue.title}"`);
                console.log(`Matching word stems: ${matchingWords.join(', ')}`);
              }

              return isMatch;
            });

            // Split issues into version-labeled and non-version-labeled
            const versionIssues = similarIssues.filter(issue => 
              issue.labels.some(label => isVersionLabel(label))
            );
            const otherIssues = similarIssues.filter(issue => 
              !issue.labels.some(label => isVersionLabel(label))
            );

            console.log(`Found ${versionIssues.length} version-labeled and ${otherIssues.length} other similar issues`);

            if (similarIssues.length > 0) {
              let commentBody = "🔍 **Similar Issues Found**\n\n";
              commentBody += "The following issues have similar titles and may be duplicates. Please review these before proceeding.\n\n";

              if (versionIssues.length > 0) {
                commentBody += "### 📌 Version-Specific Issues\n\n";
                commentBody += createIssueTable(versionIssues, true);
                commentBody += "\n";
              }

              if (otherIssues.length > 0) {
                commentBody += "### 📑 Other Related Issues\n\n";
                commentBody += createIssueTable(otherIssues, false);
                commentBody += "\n";
              }

              commentBody += "\n⚠️ Please review these similar issues before proceeding!";

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
