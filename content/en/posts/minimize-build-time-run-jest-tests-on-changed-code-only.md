---
title: "Minimize Build Time: How to run Jest Tests on Changed Code Only"
date: 2023-09-04
draft: false
tags:
  - React
  - Testing
  - Automation
authors:
  - tomas-panik
summary:
  Efficiency is a cornerstone of any CI/CD pipeline. Running the full suite of Jest tests for every small modification can be a resource-intensive task that slows down the development process. We will learn how to run only those Jest tests that are impacted by the changes in a GitHub Pull Request.

---
## Introduction

Efficiency is a cornerstone of any CI/CD pipeline. Running the full suite of Jest tests for every small modification can be a resource-intensive task that slows down the development process. Normally, your `package.json` might contain a script to run tests like this:

```json
{
  "scripts": {
    "test:ci": "jest --watchAll=false --silent --reporters=default --reporters=jest-junit"
  }
}
```

To optimize this, we can create a test running script called `run-tests.js`. This script will allow us to run only those Jest tests that are impacted by the changes in a GitHub Pull Request. This approach, while demonstrated on CircleCI, is designed to be agnostic of the build pipeline infrastructure.

## 1. Importing Required Modules

The first step in our script is to import the modules needed for HTTP requests, process spawning, and path manipulation:

```javascript
const fetch = require('node-fetch');
const { spawn } = require('child_process');
const path = require('path');
```

## 2. Jest Configuration and Function to Run Jest

The Jest command and its arguments are set up in the following block. This includes base Jest command that needs to run:

```javascript
const codeRoot = '.'; // in case your package.json is not in the repo root put the path here
const sourceCodeBase = 'src';
const jestCmd = 'jest';
const jestArgs = [
  '--watchAll=false',
  '--silent',
  '--reporters=default',
  '--reporters=jest-junit',
];
```

We then define a function called `runJest` that will execute the Jest tests with these arguments:

```javascript
const runJest = (args) => {
  return new Promise((resolve, reject) => {
    const jestProcess = spawn(jestCmd, args, { stdio: 'inherit', shell: true });
    jestProcess.on('close', (code) => {
      if (code === 0) {
        resolve();
      } else {
        reject(new Error(`Jest exited with code ${code}`));
      }
    });
  });
};
```

## 3. Fetching Changed Files

The next function, `fetchChangedFiles`, communicates with GitHub's API to determine which files have been altered in the pull request:

```javascript
const fetchChangedFiles = async (repo, prNumber, token) => {
  const url = `https://api.github.com/repos/${repo}/pulls/${prNumber}/files`;
  const response = await fetch(url, {
    headers: {
      Accept: 'application/vnd.github+json',
      Authorization: `Bearer ${token}`,
    },
  });

  if (!response.ok) {
    throw new Error(`Failed to fetch changed files: ${response.statusText}`);
  }

  const files = await response.json();
  return files.map((file) =>
    path.relative(codeRoot, file.filename)
  );
};
```

## 4. Decision Logic, which Tests to run

Depending on the files that have changed, we decide to either run all tests or only the impacted ones:

```javascript
const runImpactedTests = async (repo, prNumber, token) => {
    let changedFiles;
    try {
      changedFiles = await fetchChangedFiles(repo, prNumber, token);
    } catch (error) {
      console.log(
        'Error while fetching PR diff, running all tests:',
        error.message
      );
      return runAllTests();
    }

    console.log('Changed files:', changedFiles);

    const otherChanges = changedFiles.some((file) => !file.startsWith(sourceCodeBase));

    if (otherChanges) {
      console.log(`Changes detected outside of ${sourceCodeBase}, running all tests`);
      return runAllTests();
    } else {
      console.log('Running only impacted tests');
      return runJest([...jestArgs, '--findRelatedTests', ...changedFiles]);
    }
};
const runAllTests = () => {
    console.log('Running all tests in main-client');
    return runJest([...jestArgs]);
};
```

## 5. Main Function and Environment Variables

The `main` function combines all the above functionalities:

```javascript
const main = async () => {
  const prUrl = process.env.CIRCLE_PULL_REQUEST;
  const prNumber = prUrl ? prUrl.split('/').pop() : null;
  const repo =
    process.env.CIRCLE_PROJECT_USERNAME && process.env.CIRCLE_PROJECT_REPONAME
      ? `${process.env.CIRCLE_PROJECT_USERNAME}/${process.env.CIRCLE_PROJECT_REPONAME}`
      : null;
  const token = process.env.GITHUB_TOKEN;

  if (!prNumber || !repo || !token) {
    console.log(
      'PR number, repository information, or GitHub token is not available.'
    );
    await runAllTests();
    return;
  }

  const isPullRequest =
    process.env.CI_PULL_REQUEST && process.env.CI_PULL_REQUEST !== 'false';
  if (isPullRequest) {
    await runImpactedTests(repo, prNumber, token);
  } else {
    await runAllTests();
  }
};
```

## 6. Script Execution and Error Handling

Finally, the script calls the `main` function and handles any potential errors:

```javascript
(async () => {
  try {
    await main();
  } catch (error) {
    console.error('Error while running tests:', error.message);
    process.exit(1);
  }
})();
```

## The Full Script

After dissecting each part, here's the complete `run-tests.js` script:

```javascript
const fetch = require('node-fetch');
const { spawn } = require('child_process');
const path = require('path');

const codeRoot = '.'; // in case your package.json is not in the repo root put the path here
const sourceCodeBase = 'src';

// Jest configuration
const jestCmd = 'jest';
const jestArgs = [
  '--watchAll=false',
  '--silent',
  '--reporters=default',
  '--reporters=jest-junit',
];

// Run Jest with the provided arguments
const runJest = (args) => {
  return new Promise((resolve, reject) => {
    const jestProcess = spawn(jestCmd, args, { stdio: 'inherit', shell: true });

    jestProcess.on('close', (code) => {
      if (code === 0) {
        resolve();
      } else {
        reject(new Error(`Jest exited with code ${code}`));
      }
    });
  });
};

// Fetch the list of changed files in a pull request
const fetchChangedFiles = async (repo, prNumber, token) => {
  const url = `https://api.github.com/repos/${repo}/pulls/${prNumber}/files`;
  const response = await fetch(url, {
    headers: {
      Accept: 'application/vnd.github+json',
      Authorization: `Bearer ${token}`,
    },
  });

  if (!response.ok) {
    throw new Error(`Failed to fetch changed files: ${response.statusText}`);
  }

  const files = await response.json();
  return files.map((file) =>
    path.relative(codeRoot, file.filename)
  );
};

// Run all tests in main-client
const runAllTests = () => {
  console.log('Running all tests');
  return runJest([...jestArgs]);
};

// Run only the tests impacted by the changes in the pull request
const runImpactedTests = async (repo, prNumber, token) => {
  let changedFiles;
  try {
    changedFiles = await fetchChangedFiles(repo, prNumber, token);
  } catch (error) {
    console.log(
      'Error while fetching PR diff, running all tests:',
      error.message
    );
    return runAllTests();
  }

  console.log('Changed files:', changedFiles);

  const otherChanges = changedFiles.some((file) => !file.startsWith(sourceCodeBase));

  if (otherChanges) {
    console.log(`Changes detected outside of ${sourceCodeBase}, running all tests`);
    return runAllTests();
  } else {
    console.log('Running only impacted tests');
    return runJest([...jestArgs, '--findRelatedTests', ...changedFiles]);
  }
};

// Main function to determine whether to run all tests or only impacted tests
const main = async () => {
  const prUrl = process.env.CIRCLE_PULL_REQUEST;
  const prNumber = prUrl ? prUrl.split('/').pop() : null;
  const repo =
    process.env.CIRCLE_PROJECT_USERNAME && process.env.CIRCLE_PROJECT_REPONAME
      ? `${process.env.CIRCLE_PROJECT_USERNAME}/${process.env.CIRCLE_PROJECT_REPONAME}`
      : null;
  const token = process.env.GITHUB_TOKEN;

  if (!prNumber || !repo || !token) {
    console.log(
      'PR number, repository information, or GitHub token is not available.'
    );
    await runAllTests();
    return;
  }

  const isPullRequest =
    process.env.CI_PULL_REQUEST && process.env.CI_PULL_REQUEST !== 'false';
  if (isPullRequest) {
    await runImpactedTests(repo, prNumber, token);
  } else {
    await runAllTests();
  }
};

// Execute the main function and handle errors
(async () => {
  try {
    await main();
  } catch (error) {
    console.error('Error while running tests:', error.message);
    process.exit(1);
  }
})();
```

## Conclusion

We've created a script that selectively runs Jest tests based on the code changes in a GitHub Pull Request. This not only makes your CI/CD pipeline more efficient but also speeds up your development cycle.

Do you have thoughts or questions? Maybe a different approach to optimizing test runs? 

Feel free to leave a comment below.
