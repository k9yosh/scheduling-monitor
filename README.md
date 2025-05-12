# Scheduling Job Monitor Frontend

This project is a Vue.js frontend application designed to monitor and interact with a Spring Boot backend that manages scheduled jobs (likely using Spring Batch).

It provides a real-time dashboard displaying active job executions and a history of recent jobs, along with a control panel to launch new job simulations.

## Features

*   **Control Panel:** Launch predefined job simulations with configurable durations.
*   **Real-time Job Dashboard:** 
    *   Displays currently running/starting jobs as cards.
    *   Uses Server-Sent Events (SSE) for real-time status updates.
    *   Includes visual indicators (blinking status badge) for active jobs.
    *   Finished jobs remain visible for a short duration before animating out.
*   **Job History:** Displays recently completed, failed, or stopped jobs in a table below the active jobs.

## Prerequisites

*   **Node.js:** A recent LTS version is recommended (e.g., v20.x or later).
*   **npm:** Version compatible with your Node.js installation (usually included).
*   **Backend Server:** The corresponding Spring Boot application ([`schedule-job-runner`](https://github.com/k9yosh/schedule-job-runner)) must be running and accessible at `http://localhost:8080`.

## Installation

1.  Clone the repository (if applicable).
2.  Navigate to the project directory in your terminal.
3.  Install the dependencies:
    ```bash
    npm install
    ```

## Running the Application

1.  **Ensure the backend Spring Boot application is running.**
2.  Start the Vue development server:
    ```bash
    npm run dev
    ```
3.  Open your web browser and navigate to the URL provided by Vite (usually `http://localhost:5173` or similar).

## Project Structure

*   `public/`: Static assets.
*   `src/`: Application source code.
    *   `assets/`: Static assets processed by Vite.
    *   `components/`:
        *   `ControlPanel.vue`: UI for launching jobs.
        *   `JobDashboard.vue`: UI for displaying active jobs and history.
    *   `App.vue`: Main application layout component.
    *   `main.ts`: Application entry point.
*   `index.html`: Main HTML file.
*   `package.json`: Project dependencies and scripts.
*   `vite.config.ts`: Vite configuration.
*   `tsconfig.json`: TypeScript configuration.
