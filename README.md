# node-app repo for deploying  a sample app docker image, Codebuild to ECR (CICD/ECS)

Clone my git
git clone https://github.com/haghmiuni/node-app.git

cd node-app

docker build -t my-custom-nginx .
docker images
docker run --name my-custom-nginx-container -p 80:8080 -d my-custom-nginx
docker ps -a

try browsing to localhost:80
