# 전체 아키텍처

> **본 Hands on Lab은 SEOUL region( ap-northeast-2 ) 을 기준으로 작성되었습니다. 실습을 하실 때 반드시 리전을 확인하기 바랍니다. 또한 DemoGo - Amazon ECS Cats and Dogs 환경을 그대로 사용하므로, 본 실습 전에 선행 조건으로써 DemoGo Fargate/ECR 환경이 구성되어 있어야 합니다.**

본 HOL 통해서 아래와 같은 아키텍처를 구성합니다. 이번 실습을 통하여 Code Pipeline으로 파이프라인을 생성하여 지속적으로 소스를 빌드하고 배포합니다.

![Alt](images/overall-architecture.png "overall architecture")

# CodeCommit 리포지토리 생성하기

1. 다음의 링크에서 [https://console.aws.amazon.com/codesuite/codecommit/home](https://console.aws.amazon.com/codesuite/codecommit/home) 에서 CodeCommit 콘솔을 엽니다.

2. Repositories 페이지에서 Create repository를 선택합니다.

3. 리포지토리 생성 페이지의 Name에 **dogs**  등과 같이 입력합니다. 이미 존재하는 repository면 다른 이름을 넣어서 생성합니다.

4. (선택 사항) 설명에 리포지토리 설명을 입력합니다. 그러면 사용자들이 리포지토리의 용도를 식별하는 데 도움이 됩니다.

5. Create를 버튼을 클릭하여 리포지토리를 생성합니다.

## Code Commit에서 사용할 HTTPS Git Credential(자격증명) 생성하기

1. AWS Management 콘솔에 로그인한 [다음 링크를 클릭해서 IAM 콘솔을 엽니다](https://console.aws.amazon.com/iam/). CodeCommit 접속을 위해 Git 자격 증명을 생성 및 사용할 IAM 사용자로 로그인해야 합니다.

2. IAM 콘솔의 탐색 창에서 Users(사용자)를 선택하고 사용자 목록에 기존에 작성하셨던 유저명 혹은 **Administrator** 을 선택합니다.

3. AWS CodeCommit 자격 증명 탭을 선택합니다. 그리고 하단에 있는 Generate Git Credentials 버튼을 클립합니다.

     ![Alt](./images/generate-git-credential.png "generate git credential")

4. IAM이 생성한 사용자 이름과 암호를 복사하는 방법은 로컬 컴퓨터에 있는 안전한 파일에 표시, 복사 후 붙여넣기하거나 자격 증명 다운로드를 선택하여 .CSV 파일로 이 정보를 다운로드하는 두 가지가 있습니다. CodeCommit에 접속하려면 이 정보가 필요합니다. Download credentials 버튼을 눌러서 CSV 파일로 저장해두록 합니다.

    ![Alt](./images/download-git-credential.png "generate git credential")

     > **이때가 사용자 이름과 암호를 저장할 수 있는 유일한 기회입니다.** 이 정보를 저장하지 않는 경우, 사용자 이름은 IAM 콘솔에서 복사할 수 있지만 암호는 찾을 수 없습니다. 그러므로 암호를 재설정한 후 저장해야 합니다.

## CodeCommit 콘솔 연결 및 리포지토리 복제

1. [https://console.aws.amazon.com/codesuite/codecommit/home](https://console.aws.amazon.com/codesuite/codecommit/home) 에서 CodeCommit 콘솔을 엽니다.

2. 오른쪽 상단에서 리전 선택메뉴에서 Seoul 리전을 선택합니다. 리포지토리는 한 AWS 리전에 국한됩니다.

3. 목록에서 연결하려는 리포지토리명을 클릭합니다. 그러면 해당 리포지토리의 코드 페이지가 열립니다.

4. 우측 상단의 Clone URL > Clone HTTPs 버튼을 눌러서 Git 리포지토리 URL을 복사합니다.

    ![Alt](./images/copy-codecommit-repo-url.png "generate git credential")

5. Amazon EC2 Workstation 의 터미널 화면으로 이동 후, 아래 명령어로 git 를 설치합니다.

	```bash
	sudo yum install -y git
	```
	
6. 사용자의 홈 디렉토리 아래 dogs 디렉토리를 만들고 해당 디렉토리로 이동합니다. 터미널에서 아래의 명령어를 실행합니다.

     ```bash
     cd ~
     ```

7. 터미널 화면에서 위의 4에서 복사한 명령어를 붙여넣고 실행합니다. 그럼 다음과 같은 화면을 볼 수 있습니다.

     ```bash
     [ec2-user@ip-10-0-1-54 ~]$ git clone https://git-codecommit.ap-northeast-2.amazonaws.com/v1/repos/dogs
     Cloning into 'dogs'...
Username for 'https://git-codecommit.ap-northeast-2.amazonaws.com':
     ```

8. Code Commit에서 사용할 HTTPS Git Credential(자격증명) 생성하기 항목에서 생성한 Git Credential 파일을 열어서 User Name과 Password를 보고 터미털에 복사해서 붙여넣기 합니다. 정상적으로 입력했다면 다음과 같은 화면을 볼 수 있습니다.

     ```bash
     warning: You appear to have cloned an empty repository.
     ```

9. 정상적으로 Code Commit 리포지토리를 생성하였으며 테스트를 완료했습니다.

# Code Commit Repository에 Dockefile 및 buildspec.yaml 추가하기

## buildspec.yaml 추가하기

1. Amazon EC2 Workstation의 터미널 창에서 다음의 명령어를 실행하여 dogs 디렉토리의 파일을 확인합니다.

    ```bash
    ls -al ~/catsdogs/dogs/
    ```

2. 다음의 명령어를 실행하여 dogs 디렉토리로 이동 후 기존 파일들을 복사 해옵니다.

    ```bash
	cd ~/dogs
    cp ~/catsdogs/dogs/* ./
    ```

	복사한 파일들을 확인 합니다.

    ```bash
    ls -al
    total 12
	drwxrwxr-x 3 ec2-user ec2-user   96 Mar 31 08:48 .
	drwx------ 9 ec2-user ec2-user  272 Mar 31 08:52 ..
	-rw-rw-r-- 1 ec2-user ec2-user  343 Mar 31 08:39 default.conf
	-rw-rw-r-- 1 ec2-user ec2-user  333 Mar 31 08:39 Dockerfile
	-rw-rw-r-- 1 ec2-user ec2-user  117 Mar 31 08:48 index.html
    ```

3. 다음의 명령어를 실행하여 git repository의 상태를 확인합니다.

    ```bash
    git status
    ```

    다음과 같이 출력이 됩니다.

    ```bash
    On branch master

    No commits yet

    nothing to commit (create/copy files and use "git add" to track)
    ```

4. vi 커멘드 등으로 **buildspec.yml" 파일을 추가합니다.

    > 대소문자에 유의해서 입력합니다. 파일명은 **buildspec.yml** 입니다.

5. 다음의 내용을 복사합하여 붙여넣고 저장합니다.

    > 들여쓰기 및 띄워쓰기 간격이 아래의 텍스트와 동일하도록 입력합니다!

    ```yaml
    version: 0.2

    phases:
      pre_build:
        commands:
          - echo Logging in to Amazon ECR...
          - aws --version
          - $(aws ecr get-login --region $AWS_DEFAULT_REGION --no-include-email)
          - REPOSITORY_URI=<YOUR_ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/dogs 
          - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
          - IMAGE_TAG=${COMMIT_HASH:=latest}
      build:
        commands:
          - echo Build started on `date`
          - echo Building the Docker image...
          - docker build -t $REPOSITORY_URI:latest .
          - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
      post_build:
        commands:
          - echo Build completed on `date`
          - echo Pushing the Docker images...
          - docker push $REPOSITORY_URI:latest
          - docker push $REPOSITORY_URI:$IMAGE_TAG
          - echo Writing image definitions file...
          - printf '[{"name":"dogs","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json
    artifacts:
        files: imagedefinitions.json
    ```

6. phases -> pre_build -> commands 의 4번째 라인의 "\<YOUR_ACCOUNT_ID\>" 대신에 본인의 AWS 어카운트 ID를 입력하고 앞에서 생성한 ECR 리포지토리의 주소를 입력합니다. 다음과 같은 형식이 됩니다.

    > "- REPOSITORY_URI=270867796616.dkr.ecr.ap-northeast-2.amazonaws.com/dogs"

7. vi 커멘드 등으로 index.html 코드에 변경을 가하기 위해 다음의 내용을 복사하여 붙여넣고 저장합니다.

	```bash
    <html>
	<head>
	<title>Welcome to Cats and Dogs DemoGo</title>
	</head>
	<body>
	<h1>DemoGo Dogs</h1>
    </body>
	</html>
    ```
	
8. buildspec 및 Dockerfile 파일을 커밋한 후에 소스 리포지토리에 푸쉬 합니다. 다음을 참조하여 username과 email 주소를 설정합니다.

    - 현재 디렉토리가 Git repository 클론된 디렉토리가 맞는지 확인합니다.

    ```bash
	pwd
    cd ~/dogs/
    ```

    - git crendential을 저장하기 위하여 다음과 같이 user name과 email을 글로벌로 설정합니다.

    ```bash
    git config --global user.name "DemoGo amazon"
    git config --global user.email "example@amazon.com"
    ```

    - 파일을 추가합니다.

    ```bash
    git add .
    ```

    - 변경 내용을 커밋합니다.

    ```bash
    git commit -m "Adding build specification and docker file"
    ```

    - 커밋을 푸시합니다. 앞에서 Code Commit 리포지토리를 만들때 다운로드 받아뒀던 csv 파일을 참조하여 유저네임과 패스워드를 입력합니다.

    ```bash
    git config credential.helper store
    git push origin master
    ```

    - 다음과 같은 화면을 볼 수 있습니다.

    ```bash
    Enumerating objects: 6, done.
	Counting objects: 100% (6/6), done.
	Compressing objects: 100% (6/6), done.
	Writing objects: 100% (6/6), 1.27 KiB | 1.27 MiB/s, done.
	Total 6 (delta 0), reused 0 (delta 0)
	To https://git-codecommit.ap-northeast-2.amazonaws.com/v1/repos/dogs
	* [new branch]      master -> master
    ```

9. 다음 링크를 통해서 이동하여 정상적으로 푸쉬된 파일들을 확인해 봅니다. [https://ap-northeast-2.console.aws.amazon.com/codesuite/codecommit/repositories?region=ap-northeast-2](https://ap-northeast-2.console.aws.amazon.com/codesuite/codecommit/repositories?region=ap-northeast-2)

# Cope Pipeline을 생성하여 ECS에 지속적인 배포하기

    CodePipeline 마법사를 사용하여 파이프라인 단계를 생성하고, 소스 리포지토리를 ECS 서비스에 연결합니다.

## Code Pipeline으로 새로운 파이프라인 생성

1. [https://console.aws.amazon.com/codepipeline/](https://console.aws.amazon.com/codepipeline/) 에서 CodePipeline 콘솔을 엽니다.

2. 시작 페이지에서 Create Pipeline (파이프라인 생성) 을 선택합니다.

3. CodePipeline를 처음으로 사용하는 경우 Welcome 페이지 대신 소개 페이지가 나타납니다. 지금 시작을 선택합니다.

## 단계별로 입력하기

1. 이름 페이지의 파이프라인 이름 상자에 해당 파이프라인 이름을 입력한 후 다음 단계를 선택합니다. 이 자습서에서 파이프라인 이름은 **dogs-cicd** 입니다. 나머지 항목은 디폴트로 둡니다.

    ![Alt](./images/create-pipeline.png "create pipeline")

2. Add Source Stage에서는 다음과 같이 입력을 하고 Next버튼을 누릅니다.

    - Source provider: **AWS CodeCommit**

    - Repository name: **dogs**

    - Branch Name: **master**

    - Change detection options: **Amazon CloudWatch Events (Recommended)**

3. Add build stage에서는 다음과 같이 입력을 하고 Next 버튼을 누릅니다.
  
    - Build provider: **AWS CodeBuild**
    - Region: **Asia Pacific - (Seoul)**
    - Project name 오른쪽의 Create a new build project를 선택합니다. 빌드 프로젝트 생성시에는 다음과 같이 입력 및 선택을 하고 나머지는 디폴트로 둡니다
        - Project Name: hol-build
        - Environment Image: **Managed Image**
        - Operating System: **Ubuntu**
        - Runtime: **Standard**
        - Image: **aws/codebuild/standard:3.0**
        - **Privileged 옵션 체크**
        > Privileged 옵셥을 체크하지 않는다면 Code build에서 도커 이미지를 빌드할 수 없습니다
        - Continue to CodePipeline 버튼을 누릅니다
    - Add build stage 화면으로 돌아와 Next버튼을 누릅니다
        > 마법사가 빌드 프로젝트에 대해 codebuild-hol-build-service-role과 같은 형식의 CodeBuild 서비스 역할을 생성합니다. 이 역할 이름은 나중에 Amazon ECR 권한을 역할에 추가할 때 필요하므로 메모해 둡니다.

4. Add to deploy stage에서는 다음과 같이 입력을 합니다. 나머지는 디폴트로 남겨둡니다.
    - Deploy provider : **Amazon ECS**
    - Region: **Asia Pacific - (Seoul)**
    - Cluster name: **DEMOGO-ECS**
    - Service name: **dogs**

5. 리뷰 페이지에서 파이프라인 구성을 검토하고 Create Pipeline(파이프라인 생성)을 선택하여 파이프라인을 생성합니다.

    > 이제 파이프라인이 생성되었으며 다른 파이프라인 단계에서 이 파이프라인이 실행하려고 시도합니다. 하지만 마법사가 만든 기본 CodeBuild 역할이 buildspec.yml 파일에 포함된 모든 명령을 실행할 수 있는 권한을 갖고 있지 않으므로 빌드 단계가 실패합니다. 다음 단계에서는 빌드 단계를 위한 권한을 추가합니다.

## CodeBuild 역할에 Amazon ECR 권한을 추가하기

1. [https://console.aws.amazon.com/iam/](https://console.aws.amazon.com/iam/)에서 IAM 콘솔을 엽니다.

2. 왼쪽 탐색 창에서 역할을 선택합니다.

3. 검색란에 codebuild-를 입력하고 CodePipeline 마법사가 생성한 역할을 선택합니다. 이 핸즈온랩에서의 역할이름은 **codebuild-dogs-build-service-role**입니다.

4. Summary(요약) 페이지에서 Attach policies (정책 연결)을 선택합니다.

5. AmazonEC2ContainerRegistryPowerUser 정책 왼쪽의 상자를 선택하고 Attach policy ( 정책 연결 )을 선택합니다.

## 파이프라인 테스트하기

1. 다음의 링크로 [https://ap-northeast-2.console.aws.amazon.com/codesuite/codepipeline/pipelines?region=ap-northeast-2](https://ap-northeast-2.console.aws.amazon.com/codesuite/codepipeline/pipelines?region=ap-northeast-2) 이동하여 dogs-cicd를 선택하고 오른쪽 상단의 release changes를 선택합니다.

    ![Alt](./images/run-release.png "view service status")

2. 파이프라인이 배포까지 정상적으로 수행되면 다음고 같은 화면을 볼 수 있습니다.

    ![Alt](./images/view-result.png "view service result")

3. 정상적으로 배포되었다면 [https://ap-northeast-2.console.aws.amazon.com/ec2/v2/home?region=ap-northeast-2#LoadBalancers:sort=loadBalancerName](https://ap-northeast-2.console.aws.amazon.com/ec2/v2/home?region=ap-northeast-2#LoadBalancers:sort=loadBalancerName) 에서 demogo-alb를 선택한후에 Description 탭의 DNS name을 복사하여 웹 브라우저에 붙여넣고, Dogs 화면 클릭시 "DemoGo Dogs" 텍스트가 정상적으로 뜨는지 확인합니다. 변경사항이 반영되지 않은 경우 페이지 새로 고침 F5 을 해보시기 바랍니다.

4. 이번에는 구성된 소스 리포지토리에 대한 코드를 변경하고 커밋한 후 변경 사항을 푸시합니다. Amazon EC2 Workstation 터미널에서 index.html 소스를 다음과 같이 수정합니다.

   - dogs 소스 디렉토리로 이동합니다.
	```bash
	cd ~/dogs
	```
   - index.html을 다음과 같이 수정합니다.

    ```bash
    <html>
	<head>
	<title>Welcome to Cats and Dogs DemoGo</title>
	</head>
	<body>
	<h1>DemoGo Dogs - v2</h1>
    </body>
	</html>
    ```

5. 다음과 같은 명령어로 다시 소스를 커밋하고 푸쉬합니다.

   - 소스를 추가하고 커밋합니다.

    ```bash
    git commit -am "modify html"
    ```

   - 커밋을 푸시합니다. 앞에서 Code Commit 리포지토리를 만들때 다운로드 받아뒀던 csv 파일을 참조하여 유저네임과 패스워드를 입력합니다.
    
    ```bash
    git config credential.helper store
    git push origin master
    ```

6. 정상적으로 푸쉬가 되었다면 commit hook으로 인하여 Code Pipeline이 자동으로 실행이 됩니다.

7. [https://ap-northeast-2.console.aws.amazon.com/ec2/v2/home?region=ap-northeast-2#LoadBalancers:sort=loadBalancerName](https://ap-northeast-2.console.aws.amazon.com/ec2/v2/home?region=ap-northeast-2#LoadBalancers:sort=loadBalancerName) 로 이동하여 demogo-alb를 선택한후에 Description 탭의 DNS name을 복사하여 웹 브라우저에 붙여넣고, Dogs 화면 클릭시 "DemoGo Dogs - v2" 텍스트가 정상적으로 뜨는지 확인합니다. 변경사항이 반영되지 않은 경우 페이지 새로 고침 F5 을 해보시기 바랍니다.

실습을 완료하였습니다.

## 실습후 리소스 삭제하기 (Event Engine으로 진행했을 경우에는 생략)

> Event Engine을 통해서 실습을 한 경우에는 리소스들을 삭제할 필요가 없습니다. 이 랩을 진행하는 SA가 직접 모든 리소스들을 삭제합니다.

> 실습후 반드시 리소스들을 삭제합니다. 제대로 삭제를 하지 않는 다면 귀하의 계정으로 요금이 청구될 수 있습니다. 실습때 받은 credit이 있다면 등록을 해주시길 바랍니다.

1. ALB 삭제
2. Target Group 삭제
3. ECS 서비스 삭제
4. ECS Task Definition 삭제
5. ECS 클러스터 삭제
6. ECR 이미지 레지스트리 삭제
7. Code Pipeline 삭제
8. Code commit 리포지토리 삭제
9. Code build 프로젝트 삭제
10. VPC 삭제

