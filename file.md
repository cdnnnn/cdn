How to build and transfer
Build on Ubuntu 22.04:

pnpm build:docker
This runs scripts/build-n8n.mjs (compiles the app) followed by scripts/dockerize-n8n.mjs (builds the image via docker buildx bake) package.json:17 .

Export the image to a tarball — dockerize-n8n.mjs supports writing a tarball directly instead of loading into the local daemon, controlled by DOCKER_BUILD_TARBALL_DIR , or you can just do it manually with standard Docker CLI:

docker save n8nio/n8n:local -o n8n-image.tar
Transfer the .tar file to the Ubuntu 24.04 machine (scp, rsync, USB, etc.).

Load it on the target machine:

docker load -i n8n-image.tar  
docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n n8nio/n8n:local
This mirrors how n8n's own CI does it: .github/actions/build-n8n-docker/action.yml builds and caches image tarballs which are then restored/loaded by downstream jobs via load-n8n-docker .
