# kubernetes-concepts
kubernetes-concepts

powershell :

   aws sso login --profile devops-readonly (do in powershell)

wsl : 

    rm -rf ~/.aws
    cp -r /mnt/c/Users/sghos/.aws ~/.aws
    aws sts get-caller-identity --profile devops-readonly
