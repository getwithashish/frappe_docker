## Custom Docker Image and App Installation for Frappe/ERPNext Deployment

This documentation guides you through setting up a custom Docker image using Frappe Docker, specifying required apps via `apps.json`, and modifying `docker-compose.yml` to automate app installation for an ERPNext environment.

### 1. Prepare the `apps.json` File

Create an `apps.json` file to define the list of Frappe/ERPNext apps you want included in your custom image. Example:

```json
[
  {
    "url": "https://github.com/frappe/erpnext",
    "branch": "version-15"
  },
  {
    "url": "https://github.com/frappe/wiki",
    "branch": "master"
  }
]

```

- For private repositories, use a URL embedded with a Personal Access Token (PAT) as needed.
- Ensure any app dependencies (e.g., `erpnext` when using certain HRMS or payments modules) are listed explicitly in this array.

### 2. Encode `apps.json` as Base64

Convert your `apps.json` into a Base64 string, which you will pass as a Docker build argument:

```bash
export APPS_JSON_BASE64=$(base64 -w 0 /path/to/apps.json)
```

- To verify, decode and check the output:

```bash
echo -n ${APPS_JSON_BASE64} | base64 -d > apps-test-output.json
```

### 3. Build the Custom Docker Image

Clone the official Frappe Docker repository:

```bash
git clone https://github.com/frappe/frappe_docker
cd frappe_docker
```

Build your custom image using the appropriate Dockerfile (e.g., `images/custom/Containerfile`):

```bash
docker build \
  --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
  --build-arg=FRAPPE_BRANCH=version-15 \
  --build-arg=PYTHON_VERSION=3.11.9 \
  --build-arg=NODE_VERSION=20.19.2 \
  --build-arg=APPS_JSON_BASE64=$APPS_JSON_BASE64 \
  --tag=your-registry/frappe-custom:1.0.0 \
  --file=images/custom/Containerfile .
```

- Adjust `PYTHON_VERSION` and `NODE_VERSION` as needed for your environment.

### 4. Push the Custom Image to a Registry (if applicable)

If you plan to use your image across servers or deployments, push it to your Docker registry:

```bash
docker login
docker push your-registry/frappe-custom:1.0.0
```

### 5. Configure compose file to Use the Custom Image

In your `pwd.yml` (or inherited compose YAMLs), set the services to use your custom image by defining in the compose file:

```yaml
services:
  backend:
    image: your-registry/frappe-custom:1.0.0
    networks:
      - frappe_network
```

### 6. Modify compose Commands

In the create-site service, include the apps you specified in `apps.json`, to be installed

```yaml
command:
      - >
        wait-for-it -t 120 db:3306;
        wait-for-it -t 120 redis-cache:6379;
        wait-for-it -t 120 redis-queue:6379;
        export start=`date +%s`;
        until [[ -n `grep -hs ^ sites/common_site_config.json | jq -r ".db_host // empty"` ]] && \
          [[ -n `grep -hs ^ sites/common_site_config.json | jq -r ".redis_cache // empty"` ]] && \
          [[ -n `grep -hs ^ sites/common_site_config.json | jq -r ".redis_queue // empty"` ]];
        do
          echo "Waiting for sites/common_site_config.json to be created";
          sleep 5;
          if (( `date +%s`-start > 120 )); then
            echo "could not find sites/common_site_config.json with required keys";
            exit 1
          fi
        done;
        echo "sites/common_site_config.json found";
        bench new-site --mariadb-user-host-login-scope='%' --admin-password=admin --db-root-username=root --db-root-password=admin --install-app erpnext --install-app wiki --set-default frontend;
```

### 7. Start Your Environment

Build and start your stack as usual:

```bash
docker compose -f pwd.yml up
```

### To run on ARM64 architecture follow this instructions

After cloning the repo run this command to build multi-architecture images specifically for ARM64.

`docker buildx bake --no-cache --set "*.platform=linux/arm64"`

and then

- add `platform: linux/arm64` to all services in the `pwd.yml`
- replace the current specified versions of erpnext image on `pwd.yml` with `:latest`

Then run: `docker compose -f pwd.yml up -d`

## Final steps

Wait for 5 minutes for ERPNext site to be created or check `create-site` container logs before opening browser on port 8080. (username: `Administrator`, password: `admin`)

