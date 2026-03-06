# Shell App UI

This is a simple React application.

## Getting Started

Follow these steps to run the app locally.

### Prerequisites

-  Node.js (v14 or above recommended)
-  npm (comes with Node.js)

### Installation

#### Install the dependencies:
all dependencies are downloading via:

```bash
npm install --legacy-peer-deps --verbose
```

### Run local server
1. **Run the app in production mode:**
    - `cd` in each repo inside ./web/app directory

    ```bash
    npm run start-app
    ```

2. **Run the app in development mode:**
    - `cd` in each repo inside ./web/app directory
    
    - i)    Deploy Enviroments:
        - For local deployments
            - To deploy keycloak local on your computer
                ```bash
                cd services/auth/auth-deployment
                bash manage.sh start
                ```
            - To deploy mfe local on your computer and serve dist folder on a custom port
                ```bash
                npm run build
                serve -s dist -l <port-number>
                ```

The app will usually be available at http://localhost:3000.

