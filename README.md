# Sorry-Cypress-Test-Automation-Dashboard

A Dockerized Sorry Cypress setup for running and monitoring Cypress tests, featuring a stable dashboard with MongoDB and MinIO integration.

---

## Overview

This project sets up a **Sorry Cypress** instance using Docker Compose to run and monitor **Cypress** end-to-end tests. Sorry Cypress is an open-source alternative to Cypress Dashboard, offering a self-hosted solution for:

- **Parallel test execution**
- **Result visualization**
- **Storage of test artifacts** (e.g., screenshots and videos)

The setup includes **MongoDB** for data storage and **MinIO** for artifact storage, creating a robust QA automation environment. The project demonstrates a reliable test automation workflow, addressing integration challenges to ensure accurate test reporting.

---

## Features

- **Parallel Test Execution**: Run Cypress tests concurrently using the Sorry Cypress Director service.
- **Stable Test Reporting**: Resolved the "Cannot return null for non-nullable field InstanceStats.wallClockDuration" error by using Cypress 12.17.4.
- **Artifact Storage**: Configured MinIO to store test screenshots and videos.
- **MongoDB Integration**: Stores test results and metadata in a MongoDB database.
- **Dockerized Setup**: Fully containerized with Docker Compose for easy deployment.

---

## Prerequisites

Before you begin, ensure you have:

- 🐳 **Docker** and **Docker Compose** installed.
- 🟢 **Node.js** (version 16 or higher) and **npm** for running Cypress tests locally.
- 📚 A basic understanding of **Cypress** for writing and executing tests.

---

## Setup Instructions

Follow these steps to set up and run the Sorry Cypress dashboard:

### 1. Clone the Repository

```bash
git clone https://github.com/sabri-mejri/Sorry-Cypress-Test-Automation-Dashboard.git
cd Sorry-Cypress-Test-Automation-Dashboard
```

### 2. Configure the Docker Compose File

The `docker-compose.yml` file orchestrates all services required for Sorry Cypress. Update the environment variables:

- **MongoDB Credentials**:
  - Replace `<mongo-user>` and `<mongo-pass>` with a secure username and password.
- **MinIO Credentials**:
  - Replace `<minio-user>` and `<minio-pass>` with a secure access key and secret key.
- **Dashboard Configuration**:
  - Set `GRAPHQL_SCHEMA_URL` to your API service’s address (e.g., `http://<your-ip>:4000`).

#### Docker Compose Description

The `docker-compose.yml` defines six services for a complete Sorry Cypress environment:

- **mongo**: Runs MongoDB (version 4.4) to store test results and metadata. Persists data in `./data/data-mongo-cypress` and uses port `27017`.
- **director**: Manages test orchestration and parallel execution. Connects to MongoDB and MinIO for data and artifact uploads (screenshots/videos). Exposes port `1234`.
- **api**: Provides a GraphQL API for the dashboard to query test results. Connects to MongoDB on port `4000`.
- **dashboard**: Displays test results in a web interface at `http://<your-ip>:8080`. Communicates with the API service.
- **storage**: Runs MinIO to store test artifacts, sharing the Director’s network namespace. Saves data in `./data/data-minio-cypress` and uses ports `9000` (API) and `9090` (console).
- **createbuckets**: Initializes the `sorry-cypress` bucket in MinIO and sets public access policies.

This setup ensures seamless integration, persistent storage, and secure communication between services.

### 3. Start the Services

Launch all services in the background:

```bash
docker-compose up -d
```

- Verify containers are running: `docker-compose ps`
- Check logs for errors: `docker-compose logs <service-name>` (e.g., `dashboard`, `api`)

### 4. Access the Dashboard

Navigate to `http://<your-ip>:8080` (e.g., `http://192.168.1.18:8080`) in your browser to view the Sorry Cypress dashboard. Your projects (e.g., `first-test`) should be listed.

---

## Setting Up and Running Cypress Tests

To execute Cypress tests and report results to the dashboard, follow these steps:

### 1. Initialize a Cypress Project

If you don’t have a Cypress project, create one:

```bash
mkdir cypress-project
cd cypress-project
npm init -y
npm install cypress@12.17.4 --save-dev
npx cypress open
```

This installs **Cypress 12.17.4** (required to avoid the `wallClockDuration` error) and opens the Cypress Test Runner.

### 2. Install cypress-cloud

Install the `cypress-cloud` package for Sorry Cypress integration:

```bash
npm install cypress-cloud --save-dev
```

### 3. Configure currents.config.js

Create a `currents.config.js` file in your Cypress project root:

```javascript
module.exports = {
  projectId: '<your-project-id>', // Replace with your Sorry Cypress project ID
  recordKey: '<your-record-key>', // Replace with your Sorry Cypress record key
  cloudServiceUrl: 'http://<your-ip>:1234', // Replace with your Director URL
};
```

- **projectId**: Found in the dashboard under project settings (e.g., `first-test`).
- **recordKey**: A unique key for recording results, available in the dashboard.
- **cloudServiceUrl**: Points to the Director service (e.g., `http://192.168.1.18:1234`).

### 4. Configure cypress.config.js

Create or update `cypress.config.js`:

```javascript
const { defineConfig } = require('cypress');
const { cloudPlugin } = require('cypress-cloud/plugin');

module.exports = defineConfig({
  e2e: {
    setupNodeEvents(on, config) {
      return cloudPlugin(on, config);
    },
    baseUrl: 'http://your-app-url', // Replace with your application's URL
    specPattern: 'cypress/e2e/**/*.cy.{js,jsx,ts,tsx}', // Adjust if needed
  },
});
```

- **cloudPlugin**: Enables `cypress-cloud` reporting to Sorry Cypress.
- **baseUrl**: Set to your application’s URL.
- **specPattern**: Specifies test file locations (default: `cypress/e2e`).

### 5. Write Test Files

Add tests in `cypress/e2e`. Example (`cypress/e2e/spec.cy.js`):

```javascript
describe('Example Test', () => {
  it('Visits the homepage', () => {
    cy.visit('/');
    cy.contains('Welcome').should('be.visible');
  });
});
```

### 6. Run Tests

Execute tests with the `cypress-cloud` CLI:

```bash
npx cypress-cloud run --parallel --record --key <your-record-key> --ci-build-id <unique-build-id>
```

- `--parallel`: Runs tests across multiple machines.
- `--record`: Sends results to Sorry Cypress.
- `--key`: Your record key (from `currents.config.js`).
- `--ci-build-id`: A unique identifier (e.g., `hello-cypress`).

Example:

```bash
npx cypress-cloud run --parallel --record --key somekey --ci-build-id hello-cypress
```

### 7. View Results

Visit `http://<your-ip>:8080` to see test results, including pass/fail statuses, screenshots, and videos.

---

## Project Structure

- `docker-compose.yml`: Defines services (MongoDB, MinIO, Director, API, Dashboard) and configurations.
- `data/`: Stores MongoDB data (`data-mongo-cypress`) and MinIO artifacts (`data-minio-cypress`).
- `cypress-project/` (example): Contains Cypress tests, including `currents.config.js`, `cypress.config.js`, and `cypress/e2e` test files.

---

## Solutions to Encountered Issues

### Issue: "Cannot return null for non-nullable field InstanceStats.wallClockDuration"

- **Description**: The dashboard failed to fetch test run details due to the GraphQL query `getRun` returning `null` for the `results` field, violating the non-nullable requirement for `wallClockDuration`.

- **Solution**:

  1. Identified compatibility issues between Cypress and Sorry Cypress.

  2. Downgraded to **Cypress 12.17.4**:

     ```bash
     npm install cypress@12.17.4 --save-dev
     ```

  3. Integrated `cypress-cloud/plugin` in `cypress.config.js` for proper reporting.

  4. Ran tests with:

     ```bash
     npx cypress-cloud run --parallel --record --key somekey --ci-build-id hello-cypress
     ```

     This ensured complete `results` data was sent to the dashboard.

- **Outcome**: The dashboard now displays test results reliably without errors.

---

## Future Improvements

- 🔒 Add HTTPS support with Nginx for secure dashboard access.
- 🚀 Implement CI/CD pipelines to automate test runs and deployments.
- 🛠️ Enhance error logging in Director and API services for better debugging.

---

## Contributing

Fork this repository, submit issues, or create pull requests to enhance the setup. Contributions are welcome!

---

## Acknowledgments

- Gratitude to the **Sorry Cypress** team for their open-source solution.
- Built with ❤️ by **Sabri Mejri** for QA automation enthusiasts.
