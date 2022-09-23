# SIMPLE NUMBER STORAGE MULTICONTAINER APP

Its an app running on ECS the architecture of the app is described bellow.

TEST IT YOURSELF: http://finalproject-main-1355072970.us-east-1.elb.amazonaws.com/

INFRASTRUCTURE CODE AT: https://github.com/doblealberto/ecs-aws-infra-numbers

## IMPORTANT PARTS OF THIS REPO ARE:

## WORKFLOWS
```
on:
  push:
    branches:
      - prod
      - develop
      - staging
    paths:
        - 'nginx/**'
        - 'docker-compose.yml'
        - '!server/**'
        - '!client/**'
        - '!.github/**'
        - '!README.md'
        - '!.gitignore'
```
which allows us to push our docker images to the right site via using path finders. Notice : `nginx/**` which allows us to focus on changes into that specific folder.
## BUILD AND PUSH THE IMAGE
```
      - name: Build, tag, and push image to Amazon ECR
        id: build-image-proxy
        env:
          ECR_REGISTRY: "${{ steps.login-ecr.outputs.registry }}"
          ECR_REPOSITORY: "numbers-${{ github.ref_name }}-proxy-image"
          IMAGE_TAG: ${{ steps.generate_sha.outputs.sha }}
        working-directory: ./nginx
        run: |
          
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
          echo "::set-output name=image::$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"
```
In this step we build and push the image to the ECR repository notice how we set the route to our `ECR repository` by adding the following environment 
image 
```
          ECR_REGISTRY: "${{ steps.login-ecr.outputs.registry }}"
          ECR_REPOSITORY: "numbers-${{ github.ref_name }}-proxy-image"
          IMAGE_TAG: ${{ steps.generate_sha.outputs.sha }}
```

## NGINX CONFIGURATION

`nginx` folder which containts the configuration for our reverse proxy.
```
upstream client {
    server client:${CLIENT_PORT};
}

upstream api {
    server api:${APP_PORT};
}

server {
    listen ${LISTEN_PORT};

    location / {
        proxy_pass http://client;
    }

    location /sockjs-node {
        proxy_pass http://client;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
    }

    location /api {
        rewrite /api/(.*) /$1 break;
        proxy_pass http://api;
    }
}
```
As we see `nginx` which is one of the most remarkable reverse proxy tools sits in front of our services in order to 
"catch" the requests from the internet and redirect them in a proper maneer to it's respective service, this allows
that for example reactJS can make requests like so:

```
 const saveNumber = useCallback(
    async event => {
      event.preventDefault();

      await axios.post("/api/values", {
        value
      });

      setValue("");
      getAllNumbers();
    },
    [value, getAllNumbers]
  );
```
notice: `/api/values` route on axios request.
