---
seotitle: How to deploy an Encore app to Render
seodesc: Learn how to deploy an Encore application to Render using Docker and GitHub Actions.
title: Deploy to Render
lang: ts
---

If you prefer manual deployment over the automation offered by Encore's Platform, Encore simplifies the process of deploying your app to the cloud provider of your choice. This guide will walk you through deploying an Encore app to Render using Docker through GitHub Actions.

### Prerequisites
1. **Render Account**: Make sure you have a Render account. If not, you can [sign up here](https://render.com/).
2. **Docker Installed**: Ensure Docker is installed on your local machine, Docker is used by Encore to run databases locally. You can download it from the [Docker website](https://www.docker.com/get-started).
3. **Encore CLI**: Install the Encore CLI if you haven’t already. You can follow the installation instructions from the [Encore documentation](https://encore.dev/docs/ts/install).

### Step 1: Create an Encore App and a GitHub repository
1. **Create a New Encore App**: 
    - Create a new Encore app using the Encore CLI by running the following command:
    ```bash
    encore app create
    ``` 
    - Select the `Hello World` template.
    - Follow the prompts to create the app.

2. **Push the code to a GitHub repo**:
    - Create a new repo (public or private) on GitHub and push the code to it.
   
### Step 2: Push the Docker Image to GitHub's Container Registry
To deploy your Docker image to Render, you first need to push it to a container registry. We will be using GitHub's container registry, but you can also use DockerHub or other registries. 
Instead of pushing the image manually we will be using GitHub actions to automate the process.

1. **Create a GitHub Actions YAML file**:
   - In your repo, create a `.github/workflows/deploy-image-yaml` file with the following contents:
   
```yaml
name: Build, Push and Deploy a Docker Image to Render

on:
  push:
    branches: [ main ]

permissions:
  contents: read
  packages: write

jobs:
  build-push-deploy-image:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v5

      - name: Log in to the Container registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Download Encore CLI script
        run: curl --output install.sh -L https://encore.dev/install.sh

      - name: Install Encore CLI
        run: bash install.sh

      - name: Build Docker image
        run: /home/runner/.encore/bin/encore build docker hello-world

      - name: Tag Docker image
        run: docker tag hello-world ghcr.io/${{ github.repository }}:latest

      - name: Push Docker image
        run: docker push ghcr.io/${{ github.repository }}:latest
```

This will install the Encore CLI, build the Docker image, tag it, and push it to GitHub's container registry everytime you push to the `main` branch.
The dynamic values like `${{ github.repository }}` will be filled in automatically by GitHub, you should not need to do anything.

2. **Add, commit and push the changes**:
   - Push the changes to your GitHub repository to trigger the GitHub action.

### Step 3: Deploy the Docker image to Render

1. **Create a new service on Render**:
    - Log in to Render and go to your dashboard. 
    - Click on **"New"**.
    - Choose the **"Web Service"** option.
    - Select the "Existing Image" tab.
    - Enter the Docker Image URI, which should be something like `ghcr.io/username/repo:latest`. You can find the Docker Image under **Packages** in your GitHub repo.
    - If you created a private repository, you will need to configure a Credential to access the Docker image.
    - After clicking **Connect**, you will want to provide a **Name**, **Region**, and **Instance Type** for your service.
    - Deploy the service.

4. **Access the application**:
    - Once deployed, you will get a public URL to access your application. It should look something like this: `https://myapp.onrender.com/`.
    - Test the app to ensure it's running as expected, e.g. 
   ```bash
      curl https://myapp.onrender.com/hello/world
    ```

### Step 4: Automate the Deployment Process
Render has no way of knowing that you've pushed a new image to the container registry, but every service on Render is assigned a Deploy Hook we can call to deploy your image on a successful build.

1. **Copy your Render service's Deploy Hook**:
   - Go to your service on Render.
   - Go to **Settings** in the left hand navigation.
   - Scroll down to **Deploy Hook** in the **Deploy** section, and copy the URL.
  
2. **Add your Deploy Hook to GitHub Secrets**:
   - Go to your GitHub repository.
   - Go to **"Settings"**.
   - Click on **"Secrets and variables" → "Actions"**.
   - Click on **"New repository secret"**.
   - Add a new secret called `DEPLOYHOOK` and paste the URL you copied earlier.

3. **Add new steps to the GitHub Actions YAML file**:
   - At the bottom of the existing file, add the following steps to call the script:
```yaml
      - name: Trigger Render deployment
        run: curl ${{ secrets.DEPLOYHOOK }}
```

Whenever you push a new Docker Image to the container registry, the GitHub action will trigger a new deployment on Render.

### Step 5: Add a Database to Your App

Render provides managed databases, allowing you to add a database to your app easily. Here’s how to set up a database for your app:

1. **Create a database for your app on Render**:
   - Navigate to your Render dashboard.
   - In the upper left hand corner, press the ** + New** dropdown.
   - Select **Postgres** from the dropdown

2. **Copy the connection details**:
   - Click on the Postgres service you just created.
   - Scroll to the bottom of the **Connections** section on the **Info** page. 
   - Copy the **PSQL Command**. 
   
3. **Create a database table**:
   - Connect to the database using the `psql` command:
   ```bash
   PGPASSWORD=<password> psql -h <hostname>.<region>.render.com -U mydb_user mydb
   ```
   - Create a table
   ```sql
     CREATE TABLE users (
        id SERIAL PRIMARY KEY,
        name TEXT
     );
     INSERT INTO users (name) VALUES ('Alice');
     INSERT INTO users (name) VALUES ('Bob');
     INSERT INTO users (name) VALUES ('Carol');
     INSERT INTO users (name) VALUES ('Eve');
     INSERT INTO users (name) VALUES ('Grace');
   ```
   
4. **Declare a Database in your Encore app**:
   - Open your Encore app’s codebase.
   - Add `mydb` database to your app ([Encore Database Documentation](https://encore.dev/docs/ts/primitives/databases))
   ```typescript
      const mydb = new SQLDatabase("mydb", {
         migrations: "./migrations",
      });

      export const getUser = api(
        { expose: true, method: "GET", path: "/names/:id" },
        async ({id}: {id:number}): Promise<{ id: number; name: string }> => {
          return await mydb.queryRow`SELECT * FROM users WHERE id = ${id}` as { id: number; name: string };
        }
      );
   ```

5. **Set Up Environment Variables**:
   - Navigate to your **myapp** Web Service.
   - Under **Manage** in the left-hand navigation select **Environment**
   - In the **Environment Variables** card, press the **+ Add** drop down and select **New Variable**.
   - Add the database password as an environment variable called `DB_PASSWORD`.

6. **Create an Encore Infrastructure config**
   - Create a file named `infra.config.json` in the root of your Encore app.
   - Copy the **Hostname**, **Database**, and **Username** from the **Connections** section of your Postgres service's **Info** page.
   - Add the connection details to the file:
   ```json
   {
      "$schema": "https://encore.dev/schemas/infra.schema.json",
      "sql_servers": [
      {
         "host": "<hostname>",
         "tls_config": {
            "disable_ca_validation": true
         },
         "databases": {
            "mydb": {
               "name": "<database>",
               "username": "<username>",
               "password": {"$env": "DB_PASSWORD"}
             }
         }
      }]   
   }
   ```
   Render internal database connections do not use TLS, so we disable the CA validation.

7. **Update the **Build Docker image** step in your GitHub Actions YAML file:
   - Substitute the following for the existing **Build Docker image** section:
   ```yaml 
         - name: Build Docker image
           run: /home/runner/.encore/bin/encore build docker --config=infra.config.json hello-world
   ```
8. **Make a new deployment**:
   - Add, commit, and push the changes to your GitHub repository. This will trigger a new image build and deploy to Render.

9. **Test the Database Connection**:
   - Test the database connection by calling the API
   ```bash
    curl https://myapp.onrender.com/names/1
   ```

### Conclusion
That’s it! You’ve successfully deployed an Encore app to Render using Docker. You can now scale your app, monitor its performance, and manage it easily through the Render dashboard. If you encounter any issues, refer to the Render documentation or the Encore community for help. Happy coding!
