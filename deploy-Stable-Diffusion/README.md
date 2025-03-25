Steps of how to build stable diffusion UI image:
1. clone https://github.com/xiaoping8385/stable-diffusion-webui-docker.git 
2. copy Dockfile to stable-diffusion-webui-docker/AUTOMATIC1111/
3. copy entrypoint.sh to stable-diffusion-webui-docker/AUTOMATIC1111/
4. docker build -t 970547376847.dkr.ecr.us-west-2.amazonaws.com/sd-auto:v9 ./services/AUTOMATIC1111/ (you need change to your own aws ECR)
5. docker push 970547376847.dkr.ecr.us-west-2.amazonaws.com/sd-auto:v9