---
title: "Minimalizujte dobu buildu: Jak spustit Jest testy pouze na změněném kódu"
date: 2023-09-02
draft: true
tags:
  - React
  - Testing
  - Automation
authors:
  - tomas-panik
summary:
  Efektivita je základním kamenem jakéhokoliv CI/CD pipeline. Spouštění kompletní sady Jest testů při každé malé úpravě může být náročným úkolem, který zpomaluje vývojový proces. Naučíme se, jak spustit pouze ty Jest testy, které jsou ovlivněny změnami v GitHub Pull Requestu.

---

## Úvod

Efektivita je základním kamenem jakéhokoliv CI/CD pipeline. Spouštění kompletní sady Jest testů při každé malé úpravě může být náročným úkolem, který zpomaluje vývojový proces. Obvykle by váš `package.json` mohl obsahovat následující skript pro spuštění testů:

```json
{
  "scripts": {
    "test:ci": "jest --watchAll=false --silent --reporters=default --reporters=jest-junit"
  }
}
```

Pro optimalizaci můžeme vytvořit skript pro spuštění testů nazvaný `run-tests.js`. Tento skript nám umožní spustit pouze ty Jest testy, které jsou ovlivněny změnami v GitHub Pull Requestu. Tento přístup, ačkoli demonstrovaný na CircleCI, je nezávislý na buildovací infrastruktuře.

## 1. Importování potřebných modulů

Prvním krokem v našem skriptu je import modulů potřebných pro HTTP požadavky, spouštění procesů a manipulaci s cestami.

```javascript
const fetch = require('node-fetch');
const { spawn } = require('child_process');
const path = require('path');
```

## 2. Konfigurace Jest a funkce pro spuštění Jest

V následujícím bloku nastavujeme Jest příkaz a jeho argumenty. To zahrnuje základní Jest příkaz, který je potřeba spustit:
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

Poté definujeme funkci nazvanou `runJest`, která bude spouštět Jest testy s těmito argumenty.

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

## 3. Načítání změněných souborů

Další funkce, `fetchChangedFiles`, komunikuje s GitHub API, aby určila, které soubory byly změněny v pull requestu.

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

## 4. Rozhodovací logika, které testy spustit

V závislosti na souborech, které se změnily, se rozhodneme, zda spustit všechny testy, nebo pouze ty ovlivněné.

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

## 5. Hlavní funkce a proměnné prostředí

Funkce `main` kombinuje všechny výše uvedené funkcionality.

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

## 6. Spuštění skriptu a ošetření chyb

Nakonec skript volá funkci `main` a ošetřuje případné chyby.

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

## Kompletní skript

Zde je kompletní `run-tests.js` skript:

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

## Závěr

Vytvořili jsme skript, který selektivně spouští Jest testy na základě změn kódu v GitHub Pull Requestu. Tím nejen zefektivníte váš CI/CD pipeline, ale také urychlíte váš vývojový cyklus.

Máte nějaké myšlenky nebo otázky? Možná jiný přístup k optimalizaci běhu testů? 

Neváhejte zanechat komentář níže.