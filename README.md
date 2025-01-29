# Version-Check Repository

This repository is used to track versions of multiple projects. Each project will have its own JSON file that tracks its version.

## Table of Contents

1. [Setting Up the Version-Check Repository](#setting-up-a-version-check-repository)
2. [Adding a New Project for Version Tracking](#adding-a-new-project-for-version-tracking)
3. [Requirements for the New Project Repository](#requirements-for-the-new-project-repository)
4. [Running the Release Script](#running-the-release-script)

## Setting Up A Version-Check Repository

1. **Create a new repository** for version tracking.

   ```sh
   git init version-check
   cd version-check
   ```
Create an initial version file for the base project.

```sh
touch [project-name]-version.json
```

Add the following content to [project-name]-version.json:

```json
{
  "latest_version": "0.0.0"
}
```
Commit and push the initial version file:

```sh
git add [project-name]-version.json
git commit -m "Initial version file"
git push origin main
```

## Adding a New Project for Version Tracking
Navigate to the version-check repository:

```sh
cd version-check
```
Create a new JSON file for the new project. Replace new-project with the name of your project.

```sh
touch new-project-version.json
```

Add the following content to new-project-version.json:

```json
{
  "latest_version": "0.0.0"
}
```
Commit and push the new version file:

```sh
git add new-project-version.json
git commit -m "Add version file for new-project"
git push origin main
```
## Requirements for the New Project Repository

Initialize a new Git repository for the new project.

```sh
mkdir new-project
cd new-project
git init
```
Add the version-check repository as a submodule:

```sh
git submodule add <version-check-repo-url> version-check
git submodule update --init --recursive
```
Replace <version-check-repo-url> with the URL of the version-check repository.

Create a dev branch in the new project repository:

```sh
git checkout -b dev
git push origin dev
```
Create a version.js file in the root of the new project:

```sh
touch version.js
```
Add the following content to version.js:

```javascript
window.majorVersion = 0;
window.minorVersion = 0;
window.patchVersion = 0;
window.version = "0.0.0";
```
Create a release.js file in the root of the new project:

```sh
touch release.js
```
Add the following content to release.js:

```javascript
// SET THESE PARAMETERS BEFORE USING THIS SCRIPT!!! (see README.md)
// | | | | | | | | | | | | | | | | | | | | | | | | | | | | | | |
// V V V V V V V V V V V V V V V V V V V V V V V V V V V V V V

// version-check submodule repo name
const repoName = 'version-check';
// Set the name of project json file in the repoName submodule
const projectName = 'new-project';
// Location of the package.json file for node projects
const packageLoc = '../package.json';
// Use this to enable/disable Git commands for testing purposes
const isGitEnabled = true;

// Script Execution
const { execSync } = require('child_process');
const { readFileSync, writeFileSync, appendFileSync, existsSync } = require('fs');
const path = require('path');
const readline = require('readline');

// Create a custom logger
const logFile = path.join(__dirname, 'release.log');
const logger = {
  log: (message) => {
    console.log(message);
    appendFileSync(logFile, `${message}\n`, 'utf8');
  },
  error: (message) => {
    console.error(message);
    appendFileSync(logFile, `ERROR: ${message}\n`, 'utf8');
  }
};

// Clear the log file at the start of each run
writeFileSync(logFile, '', 'utf8');

// Function to execute a shell command and output the result
function execCommand(command) {
  try {
    logger.log(`Executing: ${command}`);
    const output = execSync(command, { encoding: 'utf8', stdio: ['inherit', 'pipe', 'pipe'] });
    logger.log(output);
    return output;
  } catch (error) {
    logger.error(`Error executing command: ${command}`);
    if (error.stdout) logger.log(`Command stdout:\n${error.stdout}`);
    if (error.stderr) logger.error(`Command stderr:\n${error.stderr}`);
    process.exit(1);
  }
}

// Update package.json version
function updatePackageJsonVersion(newVersion) {
  const packageJsonPath = path.join(__dirname, packageLoc);
  const packageJsonContent = JSON.parse(readFileSync(packageJsonPath, 'utf8'));
  if (!packageJsonContent) {
    logger.error('Error parsing package.json');
    process.exit(1);
  }
  packageJsonContent.version = newVersion;
  writeFileSync(packageJsonPath, JSON.stringify(packageJsonContent, null, 2), 'utf8');
  logger.log(`Updated package.json version to ${newVersion}`);
};

// Function to update the version in version.js and version.json
function updateVersionFile(versionType) {
  const filePath = path.join(__dirname, 'version.js');
  let fileContent = readFileSync(filePath, 'utf8');

  const majorVersionMatch = fileContent.match(/window\.majorVersion\s*=\s*(\d+);/);
  const minorVersionMatch = fileContent.match(/window\.minorVersion\s*=\s*(\d+);/);
  const patchVersionMatch = fileContent.match(/window\.patchVersion\s*=\s*(\d+);/);

  let majorVersion = parseInt(majorVersionMatch[1]);
  let minorVersion = parseInt(minorVersionMatch[1]);
  let patchVersion = parseInt(patchVersionMatch[1]);

  if (versionType === 'major') {
    majorVersion++;
    minorVersion = 0;
    patchVersion = 0;
  } else if (versionType === 'minor') {
    minorVersion++;
    patchVersion = 0;
  } else if (versionType === 'patch') {
    patchVersion++;
  } else {
    logger.error('Invalid version type.');
    process.exit(1);
  }

  const newVersion = `${majorVersion}.${minorVersion}.${patchVersion}`;

  // Update version.js content
  fileContent = fileContent.replace(/window\.majorVersion\s*=\s*\d+;/, `window.majorVersion = ${majorVersion};`)
    .replace(/window\.minorVersion\s*=\s*\d+;/, `window.minorVersion = ${minorVersion};`)
    .replace(/window\.patchVersion\s*=\s*\d+;/, `window.patchVersion = ${patchVersion};`)
    .replace(/window\.version\s*=\s*.*;/, `window.version = "${newVersion}";`);
  writeFileSync(filePath, fileContent, 'utf8');
  logger.log(`Updated version.js to version ${newVersion}`);

  // Update repoName submodule before writing changes
  if (isGitEnabled) {
    execCommand(`git -C ${repoName} pull origin main`);
  }

  // Update version.json content in the repoName submodule
  const jsonFilePath = path.join(__dirname, `${repoName}/${projectName}-version.json`);
  const jsonContent = JSON.stringify({
    latest_version: newVersion
  }, null, 2);
  writeFileSync(jsonFilePath, jsonContent, 'utf8');
  logger.log(`Updated version.json to version ${newVersion}`);

  return newVersion;
}

// Prompt user for the version type
const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout
});

// Wrap the question in a promise to log the user's input
function askQuestion(query) {
  return new Promise(resolve => {
    rl.question(query, (answer) => {
      logger.log(`User input: ${answer}`);
      resolve(answer);
    });
  });
}

// Main execution
(async () => {
  try {
    const versionType = await askQuestion('Is this a patch, minor, or major release? ');
    const validTypes = ['patch', 'minor', 'major'];
    if (!validTypes.includes(versionType)) {
      logger.error('Invalid version type. Please specify "patch", "minor", or "major".');
      process.exit(1);
    }

    // Update the version in the file and get the new version string
    const newVersion = updateVersionFile(versionType);

    // check if a package.json exists
    if (!existsSync(packageLoc)) {
      logger.error('No package.json found...proceeding with rest of script...');
    } else {
      updatePackageJsonVersion(newVersion);
    }

    // Merge dev into main, tag the commit, build the project, and copy the /dist folder
    if (isGitEnabled) {
      // Step 1: Checkout the dev branch and pull the latest changes
      execCommand('git checkout dev');
      execCommand('git -c core.verbose=true pull origin dev');

      // Step 3: Add and commit the updated version.json in the submodule
      execCommand(`git -C ${repoName} add -v ${projectName}-version.json`);
      execCommand(`git -C ${repoName} -c core.verbose=true commit -v -m "Update ${projectName}-version.json to v${newVersion}"`);
      execCommand(`git -C ${repoName} -c core.verbose=true push -v origin main`);

      // Step 4: Update the submodule reference in the parent repository
      execCommand(`git add -v ${repoName}`);
      execCommand(`git -c core.verbose=true commit -v -m "Update submodule ${repoName} to latest commit"`);
      execCommand('git -c core.verbose=true push -v origin dev');

      // Step 5: Add and commit the updated version.js and version.json
      execCommand('git add -v version.js');
      execCommand(`git -c core.verbose=true commit -v -m "Update version to v${newVersion}"`);
      execCommand('git -c core.verbose=true push -v origin dev');

      // Step 6: Checkout the main branch and pull the latest changes
      execCommand('git checkout main');
      execCommand('git -c core.verbose=true pull -v origin main');

      // Step 7: Merge the dev branch into the main branch
      execCommand('git -c core.verbose=true merge -v dev');

      // Step 8: Tag the commit on the main branch
      execCommand(`git tag v${newVersion}`);
      execCommand('git -c core.verbose=true push -v origin main --tags');
      execCommand('git checkout dev');
    }

    logger.log(`Release ${newVersion} created successfully!`);
  } catch (error) {
    logger.error('Error during release process:', error);
  } finally {
    rl.close();
  }
})();
```

Commit and push the new files to the new project repository:

```sh
git add release.js version.js version-check
git commit -m "Initial release script and version files"
git push origin main
```
## Running the Release Script
Navigate to the new project directory:

```sh
cd new-project
```
Run the release script:

```sh
node release.js
```

Follow the prompts to specify the type of release (patch, minor, major).

By following these steps, you will have set up a system for tracking versions of multiple projects using the version-check repository. Each project will have its own JSON file in the version-check repository, and the release script will automate the versioning and release process.
