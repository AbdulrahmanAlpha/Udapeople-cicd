Here is the full documentation for the udapeople project:

# udapeople 

## Overview
udapeople is a project built using:

- Circle CI - Cloud-based CI/CD service
- Amazon AWS - Cloud services
- AWS CLI - Command-line tool for AWS
- CloudFormation - Infrastrcuture as code
- Ansible - Configuration management tool 
- Prometheus - Monitoring tool

## Commands
The following commands are defined:

### install_ansible
This command installs Ansible.

```yaml
install_ansible:
    description: Install ansible 
    steps:
          - run:
              name:  install ansible 
              command: |
               sudo apt update
               sudo apt install software-properties-common -y
               sudo add-apt-repository --yes --update ppa:ansible/ansible
               sudo apt install ansible -y
```

### install_nodejs 
This command installs Node.JS.

```yaml
install_nodejs:
    description: Install nodejs 
    steps: 
          - run: 
              name:  Install Node.JS 
              command: | 
                 sudo apt update
                 sudo apt install nodejs
                 sudo apt install npm
```

### destroy-environment
This command destroys the backend and frontend CloudFormation stacks for a given workflow ID.

```yaml
destroy-environment:
  description: Destroy back-end and front-end cloudformation stacks given a workflow ID.
  parameters:
   WorkFlow-ID:
      type: string
      default: ${CIRCLE_WORKFLOW_ID:0:7}
  steps:
    - run:
       name: Destroy environments
       when: on_fail
       command: |
        aws cloudformation delete-stack --stack-name udapeople-backend-<< parameters.WorkFlow-ID>>
        aws s3 rm s3://udapeople-<<parameters.WorkFlow-ID>> --recursive
        aws cloudformation delete-stack --stack-name udapeople-frontend-<< parameters.WorkFlow-ID>> 
```

### revert-migrations 
This command reverts the last migration if it was successfully run in the current workflow.

```yaml
revert-migrations:
  description: Revert the last migration if successfully run in the current workflow.
  parameters:
   WorkFlow-ID:
      type: string
      default: ${CIRCLE_WORKFLOW_ID:0:7}     
  steps:
   - run:
      name: Revert migrations
      when: on_fail
      command: |
        SUCCESS=$(curl --insecure http://kvdb.io/${KVDB_BUCKET}/migration_<< parameters.WorkFlow-ID>>)
        if(( $SUCCESS==1 )); 
        then
          cd ~/project/backend
          npm install
          npm run migration:revert
        fi  
```

## Jobs
The following jobs are defined:

### build-frontend
This job builds the frontend. It installs dependencies and runs the build.

```yaml
build-frontend:
  docker:
    - image: cimg/node:13.8.0
  steps:
    - checkout
    - restore_cache:
        keys:  [frontend-depes]
    - run:
        name: Build front-end
        command: |
          cd frontend
          npm install 
          npm run build 
    - save_cache:
        paths: [frontend/node_modules]
        key: frontend-depes
```

### build-backend 
This job builds the backend. It installs dependencies and runs the build.

```yaml 
build-backend:
  docker:
    - image: cimg/node:13.8.0
  steps:
    - checkout
    - restore_cache:
        keys: [backend-depes]
    - run:
        name: Back-end build
        command: |
          cd backend
          npm install
          npm run build
    - save_cache:
        paths: [backend/node_modules]
        key: backend-depes
```

### test-frontend
This job runs frontend unit tests.

```yaml
test-frontend:
  docker:
    - image: cimg/node:13.8.0
  steps:
    - checkout
    - restore_cache:
        keys: [frontend-depes]
    - run:
        name: Front-end Unit test
        command: |
          cd frontend 
          npm install
          npm test  
```

### test-backend
This job runs backend unit tests.

```yaml 
test-backend:
  docker:
    - image: cimg/node:13.8.0
  steps:
    - checkout
    - restore_cache:
        keys: [backend-depes]
    - run:
        name: Back-end Unit test
    
        command: |
          cd backend
          npm install
          npm test 
```

### scan-frontend
This job scans the frontend dependencies.

```yaml   
scan-frontend:
  docker:
    - image: cimg/node:13.8.0    
  steps:
    - checkout
    - restore_cache:
        keys: [frontend-depes]
    - run:
        name: Front-end Scan 
        command: |
          cd frontend 
          npm install
          npm audit fix --force --audit-level=critical
          npm audit --audit-level=critical 
```

### scan-backend
This job scans the backend dependencies.

```yaml  
scan-backend:
  docker:
    - image: cimg/node:13.8.0
  steps:
    - checkout
    - restore_cache:
        keys: [backend-depes]
    - run:
        name: Back-end Scan 
        command: |
          cd backend
          npm install
          npm audit fix --force --audit-level=critical
          npm audit fix --force --audit-level=critical
          npm audit --audit-level=critical
```

### deploy-infrastructure 
This job deploys the backend and frontend infrastructure using CloudFormation.

```yaml 
deploy-infrastructure:
  docker:
    - image: cimg/base:stable
  steps:
    - checkout
    - run:
        name: install_awscli
        command: |
            sudo apt update
            sudo apt install awscli -y
    - run:
        name: Ensure back-end infrastructure exists
        command: |
          aws cloudformation deploy \
            --template-file .circleci/files/backend.yml \
            --tags project=udapeople \
            --stack-name "udapeople-backend-${CIRCLE_WORKFLOW_ID:0:7}" \
            --parameter-overrides ID="${CIRCLE_WORKFLOW_ID:0:7}"  
            
    - run:
        name: Ensure front-end infrastructure exist
        command: |
          aws cloudformation deploy \
            --template-file .circleci/files/frontend.yml \
            --tags project=udapeople \
            --stack-name "udapeople-frontend-${CIRCLE_WORKFLOW_ID:0:7}" \
            --parameter-overrides ID="${CIRCLE_WORKFLOW_ID:0:7}"  
            
    - run:
        name: Add back-end ip to ansible inventory
        command: |
          aws ec2 describe-instances \
          --filters "Name=tag:Name,Values=backend-${CIRCLE_WORKFLOW_ID:0:7}" \
          --query 'Reservations[*].Instances[*].PublicIpAddress' \
          --output text >> .circleci/ansible/inventory.txt
          cat .circleci/ansible/inventory.txt
    - persist_to_workspace:
        root: ~/
        paths:
          - project/.circleci/ansible/inventory.txt
    - destroy-environment 
```

### configure-infrastructure
This job configures the infrastructure using Ansible.

```yaml
configure-infrastructure: 
  docker:
    - image: cimg/base:stable
  environment:
        NODE_ENV: "local"
        VERSION: "1"
        ENVIRONMENT: "production"
        TYPEORM_CONNECTION: $TYPEORM_CONNECTION
        TYPEORM_HOST: $TYPEORM_HOST
        TYPEORM_USERNAME: $TYPEORM_USERNAME
        TYPEORM_PASSWORD: $TYPEORM_PASSWORD
        TYPEORM_DATABASE: $TYPEORM_DATABASE
        TYPEORM_PORT: $TYPEORM_PORT
        TYPEORM_ENTITIES: $TYPEORM_ENTITIES
  steps:
    - checkout
    - install_ansible
    - add_ssh_keys:
        fingerprints: ["eb:1d:54:01:87:85:02:2a:67:93:c2:05:21:c5:52:b4"]
    - attach_workspace:
        at: ~/
    - run:
        name: Configure server
        command: |
           cd .circleci/ansible
           cat inventory.txt 
           ansible-playbook -i inventory.txt configure-server.yml 
    - destroy-environment
```

### run-migrations
This job runs database migrations.

```yaml
run-migrations:
  docker:
    - image: cimg/node:13.8.0  
  steps:
    - checkout
    - run:
        name: install_awscli
        command: |
            sudo apt update
            sudo apt install awscli -y
    - run:
        name: Run migrations
        command: |
          cd backend 
          npm install
          npm run migrations > migrations_dump.txt 
    - run:
        name: Send migration results to memstash
        command
```
### deploy-frontend
This job deploys the frontend. It installs dependencies, builds the frontend, and deploys the files to S3.

```yaml  
deploy-frontend:
  docker:
    - image: amazon/aws-cli
  steps:
    - checkout
    - restore_cache:
        keys: [frontend-depes]
    - run:
        name: Install dependencies
        command: |
         yum update -y
         yum install -y tar gzip
         yum install -y python3
         curl -sL https://rpm.nodesource.com/setup_10.x |  bash -
         yum install -y nodejs
         pip3 install ansible
         pip3 install awscli
    - run:
        name: Install dependencies
        command: |
          cd frontend
          npm install
    - run:
        name: Get backend url
        command: |
          BACKENDIP=$(aws ec2 describe-instances \
          --filters "Name=tag:Name, Values=backend-${CIRCLE_WORKFLOW_ID:0:7}" \
          --query 'Reservations[*].Instances[*].PublicIpAddress' \
          --output text)
          
          echo "API_URL =http://${BACKENDIP}:3030" >> frontend/.env 
          cat frontend/.env 
    - run:
        name: Deploy frontend objects
        command: |
          cd frontend
          npm run build 
          aws s3 cp dist s3://udapeople-${CIRCLE_WORKFLOW_ID:0:7} --recursive
```  

### deploy-backend
This job deploys the backend. It packages the backend and deploys it using Ansible.

```yaml
deploy-backend:
  docker:
    - image: python:3.7-alpine3.11
  steps:
    - checkout 
    - add_ssh_keys:
        fingerprints: ["eb:1d:54:01:87:85:02:2a:67:93:c2:05:21:c5:52:b4"]
    - attach_workspace:
        at: ~/
    - restore_cache:
        keys: [frontend-depes]
    - run:
        name: Install dependencies
        command: |
          apk add --update ansible
          apk add --update nodejs npm
          apk add curl
          pip3 install awscli
    - run:
        name: Install dependencies
        command: |
          cd backend 
          npm install
    - run:
        name: Package Backend 
        command: |
          cd backend
          npm run build
          tar -czvf artifact.tar.gz dist/* package*
          cd ..
          cp backend/artifact.tar.gz .circleci/ansible/roles/deploy/files
    - run:
        name: Deploy backend
        command: |
          export TYPEORM_MIGRATIONS_DIR=./migrations
          export TYPEORM_ENTITIES=./modules/domian/**/*.entity{.ts,.js}
          export TYPEORM_MIGRATIONS=./migrations/*.ts

          
          cd .circleci/ansible
          echo "Contents  of the inventory.txt file is
```
### smoke-test
This job runs smoke tests on the frontend and backend.

```yaml
smoke-test:
  docker:
    - image: cimg/base:stable 
  steps:
    - checkout
    - run:
        name: install_awscli
        command: |
            sudo apt update
            sudo apt install awscli -y
    - install_nodejs
    - run:
        name: Frontend smoke test.
        command: |
          FRONTEND_WEBSITE=http://udapeople-${CIRCLE_WORKFLOW_ID:0:7}.s3-website.${AWS_DEFAULT_REGION}.amazonaws.com
          if curl -s $FRONTEND_WEBSITE | grep "Welcome"
          then 
            exit 0
          else 
            exit 1 
          fi
    - run:
        name: Backend smoke test.
        command: |
          BACKENDIP=$(aws ec2 describe-instances \
           --filters "Name=tag:Name, Values=backend-${CIRCLE_WORKFLOW_ID:0:7}" \
           --query 'Reservations[*].Instances[*].PublicIpAddress' \
           --output text)
  
          export API_URL=http://${BACKENDIP}:3030
          echo "${API_URL}"
          if curl -s ${API_URL}/api/status | grep "ok"
          then 
            exit 0 
          else                                                
            exit 1 
          fi
```

### cloudfront-update
This job updates the CloudFront distribution to point to the new deployment.

```yaml
cloudfront-update:
  docker:
    - image: cimg/base:stable 
  steps:
   - checkout
   - run:
      name: install_awscli
      command: |
           sudo apt update
           sudo apt install awscli -y
   - install_nodejs
   - run:
       name: Save Old Workflow ID TO Kvdb.io
       command: |
         export OLD_WORKFLOW_ID=$(aws cloudformation \
                             list-exports --query "Exports[?Name==\`WorkflowID\`].Value" \
                             --no-paginate --output text)
         echo "Old Workflow ID: $OLD_WORKFLOW_ID"
         echo CIRCLE_WORKFLOW_ID "${CIRCLE_WORKFLOW_ID:0:7}"
         curl https://kvdb.io/${KVDB_BUCKET}/old_workflow_id -d "${OLD_WORKFLOW_ID}"
   - run:
       name: Update cloudfront distribution
       command: |
         aws cloudformation deploy \
          --template-file .circleci/files/cloudfront.yml \
          --parameter-overrides WorkflowID="${CIRCLE_WORKFLOW_ID:0:7}" \
          --stack-name InitialStack 
``` 

### cleanup
This job cleans up old stacks and files.

```yaml
cleanup:
  docker:
    - image: cimg/base:stable 
  steps:
    - checkout
    - run:
        name: install_awscli
        command: |
            sudo apt update
            sudo apt install awscli -y
    - install_nodejs
    - run:
        name: Remove old stacks and files
        command: |
          export STACKS=($(aws cloudformation list-stacks \
            --query "StackSummaries[*].StackName" \
            --stack-status-filter CREATE_COMPLETE --no-paginate --output text))
          echo Stack names: "${STACKS[@]}"
          export OldWorkflowID=$(curl --insecure https://kvdb.io/${KVDB_BUCKET}/old_workflow_id)
          echo Old Workflow ID:"${OldWorkflowID}"
          if [[ "${STACKS[@]}" =~ "${OldWorkflowID}" ]]
          then
            aws s3 rm "s3://udapeople-${OldWorkflowID}" --recursive 
            aws cloudformation delete-stack --stack-name "udapeople-backend-${OldWorkflowID}"
            aws cloudformation delete-stack --stack-name "udapeople-frontend-${OldWorkflowID}"
          fi
```

## Workflows
The workflows for this project are:

```yaml 
workflows:
  default:
    jobs:
      - build-frontend
      - build-backend
      - test-frontend:
          requires: [build-frontend]
      - test-backend:
          requires: [build-backend]  
      - scan-backend:
          requires: [build-backend]
      - scan-frontend:
          requires: [build-frontend]  
      - deploy-infrastructure:
          requires: [test-frontend, test-backend, scan-frontend, scan-backend]
          filters:
            branches:
              only: [main]
      - configure-infrastructure:
          requires: [deploy-infrastructure]
      - run-migrations:
          requires: [configure-infrastructure]
      - deploy-frontend:
          requires: [run-migrations]
      - deploy-backend:
          requires: [run-migrations]
      - smoke-test:
          requires: [deploy-backend, deploy-frontend]
      - cloudfront-update:
          requires: [smoke-test]
      - cleanup:
          requires: [cloudfront-update]  
```
