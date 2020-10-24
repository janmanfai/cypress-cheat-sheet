# Cypress cheat sheet

```
These notes were written in mid-2019.
It is possible that the information is no longer up to date.
Please check the official Cypress documentation.
```

- [Setup](#setup)
- [Configuration](#configuration)
- [Aliases](#aliases)
- [Select Elements](#select-elements)
- [Stub XHR Requests](#stub-xhr-requests)
- [Environment Variables](#environment-variables)
- [TypeScript Support](#typescript-support)
- [Create Reports](#create-reports)
- [Recommendations](#Recommendations)
- [Links & Resources](#resources)
  - [Links](#links)
  - [Online Courses](#online-courses)
  - [Plugins](#plugins)

---

## Setup

Add Cypress via `npm install cypress --save-dev` or `yarn add cypress --dev` to your project and open the UI after installation with `cypress open`.
You may extend you script section inside your package.json like this:

```javascript
"cypress:open": "cypress open",
"cypress:run": "cypress run",
```

The first command will opeh the Cypress UI. The second will run Cypress in headless mode.

## Configuration

The minimum configuration inside the `cypress.json` should contain the baseUrl.
If you set the baseUrl you can omit passing the URL in methods like `cy.visit()`. It may also improve the startup time.

If you want to block requests from other hosts (e.g. from Google Analytics) add all these hosts to the `blacklistHosts` property.

```javascript
// cypress.json
{
  "baseUrl": "http://localhost:4200/",
  "blacklistHosts": "www.google-analytics.com",
}
```

## Aliases

You can define an alias for elements inside `beforeEach()` if you want to select an element in multiple tests.

```javascript
beforeEach(() => {
  cy.get(".element").as("myElement");
});
```

Now you can use the alias inside a test, prefixed with an `@`, like this:

```javascript
it("my test", () => {
  cy.get("@myElement").click();
});
```

## Select Elements

Instead of using a CSS Selector to select an element it is considered as best practice to use data-\* attributes for selecting elements. Using data-\* attributes is a more robust solution, because, unlike CSS classes, these do not normally change during development.

```html
<p data-test="myElement" class="class1 class2">Hello world!</p>
```

```javascript
cy.get("[data-test='myElement']");
```

You may create a Custom Command for selecting elements and use it inside your tests.

```javascript
// Custom Command
Cypress.Commands.add("getElByDataId", (id) => {
  cy.get(`[data-test='${id}']`);
});

// Usage
cy.getElByDataId("myElement");
```

## Stub XHR Requests

To intercept XHR-Request you can stub calls like this.

```javascript
// method, url, response
cy.route("GET", "/myRoute", []);
```

Every call that goes against `/myRoute` will return an empty array as response, as the third parameter describes the response.

Be aware that Cypress can only intercept XMLHttpRequests.

## Environment Variables

Use environment variables to avoid hard coded string (e.g. for API endpoints) inside your tests, so that you can easily change configurations.

Either your provide them via CLI

```javascript
cypress run --env host=myHost.local,apiEndpoint=http://localhost:8888/api/v1
```

or move them in an environment file:

```javascript
// cypress.env.json
{
  "host": "myHost.local",
  "apiEndpoint": "http://localhost:8888/api/v1"
}
```

You may even combine both approaches and load a dedicated configuration file that you pass via CLI. That is useful if you run your E2E tests inside a CI/CD pipeline and want to change settings depending on the branch.

First of all you need to write a helper function which is placed inside your plugin file.

```javascript
// cypress/plugins/index.js
const fs = require("fs-extra");
const path = require("path");

// get config file and return it
function getConfigurationByFile(file) {
  const pathToConfigFile = path.resolve(
    __dirname,
    "../config/stages",
    `${file}.json`
  );

  console.log(`${pathToConfigFile} is used for overriding configurations.`);
  return fs.readJson(pathToConfigFile);
}

module.exports = (on, config) => {
  // `on` is used to hook into various events Cypress emits
  // `config` is the resolved Cypress config

  // All configs that might be okay
  // This is relevant if you resolve config files based on your current git branch.
  const allowedConfigs = ["develop", "master"];

  // Passed config file
  const configFile = config.env.configFile;

  // accept a configFile value or use develop by default
  const file = allowedConfigs.includes(configFile) ? configFile : "develop";
  return getConfigurationByFile(file);
};
```

After that you need to define possible configurations.

```javascript
// cypress/config/stages/develop.json
{
  "host": "dev.mydomain.com",
  "apiEndpoint": "http://api.dev.mydomain.com/api/v1"
}

// cypress/config/stages/master.json
{
  "host": "mydomain.com",
  "apiEndpoint": "http://api.mydomain.com/api/v1"
}
```

Now you can run Cypress like that:

```bash
# Run headless Cypress
cypress run --env configFile=myConfigFile

# Open Cypress UI
cypress open --env configFile=develop
```

Or even integrate it into your CI/CD pipeline like this:

```yaml
# The branch or tag name for which project is built
cypress run --env configFile=$CI_COMMIT_REF_NAME
```

## TypeScript Support

If you want to use TypeScript, the existing code base must be adapted in some places.

Install Cypress Webpack Preprocessor as devDependency with `npm install --save-dev @cypress/webpack-preprocessor` or `yarn add --dev @cypress/webpack-preprocessor`.

Remove the existing files from your `cypress/` folder and create following files.

```javascript
// cypress/plugins/cy-ts-preprocessor.js
const wp = require("@cypress/webpack-preprocessor");
const webpackOptions = {
  resolve: {
    extensions: [".ts", ".js"],
  },
  module: {
    rules: [
      {
        test: /\.ts$/,
        loaders: ["ts-loader"],
        exclude: [/node_modules/],
      },
      {
        test: /\.(html|css)$/,
        loader: "raw-loader",
        exclude: /\.async\.(html|css)$/,
      },
      {
        test: /\.async\.(html|css)$/,
        loaders: ["file?name=[name].[hash].[ext]", "extract"],
      },
    ],
  },
};

const options = {
  webpackOptions,
};

module.exports = wp(options);
```

```javascript
// cypress/plugins/index.js
const cypressTypeScriptPreprocessor = require("./cy-ts-preprocessor");

module.exports = (on, config) => {
  // `on` is used to hook into various events Cypress emits
  // `config` is the resolved Cypress config

  // Enable TypeScript.
  on("file:preprocessor", cypressTypeScriptPreprocessor);
};
```

```javascript
// cypress/support/index.ts
import "./commands";
```

```javascript
// cypress/support/commands.ts
export function myFunction(): void {
  window.console.log("Hello World!");
}
Cypress.Commands.add("myFunction", myFunction);

// Overwrite the given namespace to make the custom commands available.
declare global {
    namespace Cypress {
        interface Chainable {
            myFunction(): typeof myFunction;
        }
    }
}
```

```javascript
// cypress/tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "baseUrl": "../node_modules",
    "target": "es2015",
    "lib": ["es2015", "dom"],
    "types": ["cypress"]
  },
  "include": [
    "**/*.ts"
  ]
}
```

Now your environment is ready and you can write your tests with TypeScript.

## Create Reports

As Cypress is based on Mocha we can generate reports e.g. with [mochawesome-report-generator](mochawesome-report-generator).

Install the necessary devDependencies with `npm install --save-dev mocha@5.2.0 mochawesome mochawesome-merge mochawesome-report-generator` or `yarn add --dev mocha@5.2.0 mochawesome mochawesome-merge mochawesome-report-generator`.
Note: It seems that using mocha^6.0.0 throws a TypeError and fails if a report should be generated. Therefore we use mocha@5.2.0 ([GitHub issue](https://github.com/cypress-io/cypress/issues/3537)).

Extend your cypress.json to define the reporter options.

```javascript
{
  "reporter": "mochawesome",
  "reporterOptions": {
    "overwrite": false,
    "html": false,
    "json": true,
    "reportDir": "cypress/reports"
  }
}
```

You can generate the reports manually.

```bash
# Merge all .json files inside cypress/reports/ into one file named cypress/reports/reports.json.
# After that create a report of this file with mochawesome-report-generator.
mochawesome-merge --reportDir cypress/reports/ > cypress/reports/reports.json && marge cypress/reports/reports.json --reportDir cypress/reports/
```

If you do not specify the reporter inside the `cypress.json` you can pass the reporter like this:

```bash
cypress run --reporter mochawesome
```

Modify the scripts section inside your package.json like this:

```javascript
  "cypress:run": "cypress run --reporter mochawesome",
  "cypress:createReport": "mochawesome-merge --reportDir cypress/reports/ > cypress/reports/reports.json && marge cypress/reports/reports.json --reportDir cypress/reports/"
```

Now you can call `npm run cypress:run && npm run cypress:createReport` or `yarn cypress:run && yarn cypress:createReport` to run the tests and create the report.
You'll find the generated report in `cypress/reports/report.html`.

## Recommendations

- Split application and test logic. Either using Page Objects or Custom Commands.
- Separate test logic and test data. Save all your test data inside `fixtures/` and load them inside the test using `cy.fixture('myFile')`.
- Focus on UI testing (with stubbed XHR) and only test the critical paths E2E.
- Do not slow down your tests with unnecessary sleeping, because Cypress automatically waits.
- Test the application the way a user would use it. If possible use ARIA or data- attributes instead of CSS Selecors.
- Assert frequently so you can reproduce the problem easier & faster.

## Resources

### Links

[Documentation](https://docs.cypress.io/guides/overview/why-cypress.html)

[Gleb Bahmutov](https://glebbahmutov.com/blog/tags/cypress/)

### Online Courses

[Test Production Ready Apps with Cypress](https://egghead.io/courses/test-production-ready-apps-with-cypress)

[End to End testing with Cypress](https://egghead.io/courses/end-to-end-testing-with-cypress)

### Plugins

[Cypress Image Snapshot](https://www.npmjs.com/package/cypress-image-snapshot)

[Percy (Visual Testing & Review)](https://docs.percy.io/docs/cypress)

[cypress-skip-and-only-ui](https://www.npmjs.com/package/cypress-skip-and-only-ui)

[cypress-watch-and-reload](https://www.npmjs.com/package/cypress-watch-and-reload)

[cypress-testing-library](https://www.npmjs.com/package/@testing-library/cypress)
