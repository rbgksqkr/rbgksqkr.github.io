---
title: "[AWS] Access Key 없이 CodeBuild + CodePipeline + Lambda Function 활용하여 배포 자동화하기"
layout: single
excerpt: Access Key 없이 배포 자동화하는 방법을 소개한다.

categories:
  - trouble-shooting
tags: [AWS, codeBuild, codePipeline, Lambda Function]
toc_label: 렌더링 최적화
toc: true
toc_sticky: true
---

> **[ 배포 자동화 흐름 요약 ]**
>
> 0. CodePipeline 으로 source, build, deploy, invalidate 단계를 생성한다.
> 1. `source`: 지정한 브랜치에 push되면 CodePipeline 이 변화를 감지하여 소스 코드를 가져온다.
> 2. `build` : CodeBuild 를 이용해 지정한 브랜치의 최신 소스 코드를 빌드한다.
> 3. `deploy` : CodePipeline 을 통해 빌드한 output을 s3에 저장한다.
> 4. `invalidate` : Lambda Function 을 활용하여 cloudfront의 캐싱을 무효화하고 최신 빌드 결과물을 User에게 제공한다.

# 📘 문제 상황

AWS ACCESS KEY를 발급받지 못하는 상황이라 github actions 를 통한 s3 + cloudfront 접근이 불가능하다. 따라서 검색한 내용으로는 아래 2가지 방식으로 cloudfront에 배포가 가능하다. 두 방법 중 **2번을 선택**하였다.

---

**[ ACCESS KEY 없이 cloudfront 배포하는 방법 ]**

1.  **EC2로 우회** - self-hosted runner 방식으로 EC2를 띄워, EC2에서 s3에 업로드하고 cloudfront 캐싱 무효화하기

    - EC2를 사용할 경우 인프라에 신경써야하는 비용이 커진다.
    - 배포 과정에서 문제가 발생했을 때, 트러블 슈팅이 어려울 것으로 예상된다.

2.  **CodeBuild + CodePipeline + Lambda Function 활용 ✅**

    - 인프라 작업을 aws에 위임하여 서비스 기능 구현에 집중할 수 있다.

# 📘 일단 수동으로 배포해보자

수동으로 우리 서비스를 cloudfront에 서빙하기 위해 했던 작업 순서를 생각해보자. 해당 작업은 어떻게 이뤄지는걸까?

> 1.  s3에 빌드 결과물을 올린다.
> 2.  cloudfront 캐싱 무효화한다.

## ✅ 1. s3에 빌드 결과물을 올린다.

로컬 환경에서 빌드를 하면 빌드 결과물이 dist 폴더에 담길 것이다. 해당 dist 폴더 파일을 모두 s3에 업로드한다.

> 1.  로컬 환경에서 애플리케이션을 빌드한다.
> 2.  빌드한 결과물을 s3에 업로드한다.
> 3.  파일과 폴더를 모두 업로드해줘야 의도한대로 동작한다.

![](https://velog.velcdn.com/images/ghenmaru/post/4deba42f-6ce5-4b30-99f5-db85e1c811a0/image.png)

## ✅ 2. cloudfront 캐싱 무효화한다.

s3에 업로드했으면 업로드한 파일을 cloudfront에게 인지시켜줘야 한다. 사용자는 s3가 아닌 cloudfront로 접근하며, 빌드 결과물은 s3에만 존재한다. 따라서 cloudfront의 캐싱을 무효화해주면 최신 s3 데이터로 cloudfront가 접근한다.

> Cloudfront > 상세보기 > 무효화 > 객체 경로에 `/*` 추가하고 무효화 생성

![](https://velog.velcdn.com/images/ghenmaru/post/21ab9ca6-5ab7-4b2f-acbf-fc61c7efb2df/image.png)

![](https://velog.velcdn.com/images/ghenmaru/post/1e935aef-ed44-4eab-aee5-187902bf7d1e/image.png)

# 📘 수동으로 배포한 흐름을 자동화해보자

위 2가지 작업을 수행하면 cloudfront로 우리 서비스에 접근할 수 있다. 그럼 이제 **2가지 작업을 자동화시키는 방법**에 대해 고민해보자.

## ✅ 1. s3에 빌드 결과물을 올린다 (CodeBuild + CodePipeline)

가상의 컴퓨터에서 동작한다고 생각하면 최신 코드를 가져오는 작업, 빌드 작업, 빌드한 결과물을 업로드하는 작업으로 총 3가지가 필요하다.

> 1.  source: 지정한 브랜치에 push되면 **`CodePipeline`** 이 변화를 감지하여 소스 코드를 가져온다.
> 2.  build : **`CodeBuild`** 를 이용해 지정한 브랜치의 최신 소스 코드를 빌드한다.
> 3.  deploy : **`CodePipeline`** 을 통해 빌드한 output을 s3에 저장한다.

## ✅ 2. cloudfront 캐싱 무효화한다. (Lambda Function + CodePipeline)

> 4. invalidate : **`Lambda Function`** 을 활용하여 cloudfront의 캐싱을 무효화하고 최신 빌드 결과물을 User에게 제공한다.

![](https://velog.velcdn.com/images/ghenmaru/post/c974c1dc-c846-40a4-aa6d-ccc7311f1564/image.png)

### 🚨**Lambda Function**을 활용하여 cloudfront 캐싱 무효화

**Lambda Function을 수행할 python 버전 : 3.8**

lambda function [python 버전 결정 이유](https://aws.amazon.com/ko/blogs/apn/comparing-aws-lambda-arm-vs-x86-performance-cost-and-analysis-2/) : 성능, 비용 측면에서 우수

[x86_64 결정 이유](https://www.amanox.ch/en/awslambda/) : x86_64는 CPU 계산, arm64는 I/O 작업에 특화되었으므로 x86_64 선택

**Lambda Function 생성 방법**

1. lambda_function.py 작성 후 deploy 클릭

```python
import json
import boto3

code_pipeline = boto3.client("codepipeline")
cloud_front = boto3.client("cloudfront")

def lambda_handler(event, context):
    job_id = event["CodePipeline.job"]["id"]
    objectPaths = ["/*"]

    try:
        cloud_front.create_invalidation(
            DistributionId='cloudfront DistributionId 기입',
            InvalidationBatch={
                "Paths": {
                    "Quantity": len(objectPaths),
                    "Items": objectPaths,
                },
                "CallerReference": event["CodePipeline.job"]["id"],
            },
        )
    except Exception as e:
        code_pipeline.put_job_failure_result(
            jobId=job_id,
            failureDetails={
                "type": "JobFailed",
                "message": str(e),
            },
        )
    else:
        code_pipeline.put_job_success_result(
            jobId=job_id,
        )
```

2. IAM role에 `put_job_failure_result`, `put_job_success_result` 권한 추가

```python
"Statement": : [
	{
    "Effect": "Allow",
    "Action": [
        "codepipeline:PutJobFailureResult",
        "codepipeline:PutJobSuccessResult",
        "cloudfront:CreateInvalidation"
    ],
    "Resource": "*"
	},
	...
]
```

3. CodePipeline 에 deploy 다음 단계로 lambda function 실행 단계 추가
   ![](https://velog.velcdn.com/images/ghenmaru/post/e39cba13-6397-4edf-a007-fcb0b2ccd952/image.png)

# 🐞 트러블 슈팅

배포 파이프라인 구축 외에도 추가적으로 `한 브랜치 내에서 frontend 디렉토리 내에 변경사항이 발생할 때만 빌드가 이뤄지도록 분기처리`, `dev환경에서는 storybook까지 확인할 수 있도록 cloudfront function 설정`을 적용하였다.

## ✅ frontend 디렉토리 내에 변경사항이 발생할 때만 빌드하기

특히 frontend 디렉토리에서만 변경사항이 발생할 때만 빌드가 이뤄지도록 처리하는 부분이 어려웠다. 찾아봤을 때는 2가지 방법이 있었다.

>

1. Lambda Function을 활용하여 빌드하기 전에 s3 버킷 데이터와 최신 커밋을 비교한다.
2. 빌드할 때 커밋 2개를 가져와 두 커밋을 비교한다.

1번 방식을 선택하면 source 단계와 build 단계 사이에서 실행하여 CodeBuild 자체가 안돌도록 설정할 수 있을 것 같은데, `버킷 get 요청 권한이 없어 불가능하였다.`

따라서 **git commit 2개를 가져와 두 커밋 사이의 변경사항을 체크하고, frontend 디렉토리에서 변경 사항이 있을 때만 빌드가 이뤄지도록 구현하였다. **

```yml
version: 0.2

phases:
  install:
    commands:
      - echo Install dependencies...
      - git init
      - git remote add origin https://github.com/woowacourse-teams/2024-ddangkong.git
      - git fetch --depth=2 origin develop
      - git checkout -f develop
      - |
        if git diff --name-only HEAD~1 HEAD | grep ^frontend/; then
          echo 'Changes detected in frontend directory. Proceeding with build...'
          echo 'Install dependencies...'
          cd frontend
          npm install
          export SHOULD_BUILD=true
        else
            echo 'No changes detected in frontend directory. Skipping build...'
            export SHOULD_BUILD=false
        fi
  build:
    commands:
      - echo Building...
      - |
        if [ "$SHOULD_BUILD" = true ]; then
          echo set environment variables...
          echo API_BASE_URL=$API_BASE_URL >> .env
          echo SENTRY_DSN=$SENTRY_DSN >> .env
          echo SENTRY_AUTH_TOKEN=$SENTRY_AUTH_TOKEN >> .env
          echo Building the React application...
          npm run build-prod
          echo Build finished...
          ls
          echo Building storybook...
          npm run build-storybook
          echo Build storybook finished...
          mv ./storybook-static ./dist/storybook
          cd dist
          ls
        else
          echo Skipping build phase...
        fi

artifacts:
  base-directory: frontend/dist
  files:
    - "**/*"
  name: ddangkong-frontend-deploy

cache:
  paths:
    - "node_modules/**/*"
```

<img src="https://velog.velcdn.com/images/ghenmaru/post/d11df94f-af52-4feb-9183-994cadb1f91a/image.png" alt='' width="500px" />

CodeBuild 자체는 돌지만 install 단계에서 비교가 진행되므로, 변경사항이 없으면 **전체 빌드 시간인 2분 30초가 아니라 10초 안에 빌드가 실패하여 불필요한 빌드 실행 시간을 줄였다.** [쓰는 만큼 비용이 드는 CodeBuild](https://aws.amazon.com/ko/codebuild/pricing/) 요금제에 따르면 유의미한 방식이라고 생각한다. 설정하지 않으면 서버 코드가 머지되도 불필요한 빌드가 2분씩 돌았을 것이다.

지금 보면 왜그렇게 시간이 오래걸렸는지 모르겠는데 처음 해보는 방식이였기 때문에 많은 삽질이 이뤄졌고, 그러한 삽질 덕분에 더 잘 기억에 남았다고 생각한다.

## reference

- [ec2 self-hosted runner로 배포하기](https://velog.io/@shackstack/s3-cicd-without-accesskey)
- [codeBuild + codePipeline 배포 자동화](https://medium.com/fullstackai/aws-creating-a-cloudfront-invalidation-in-codepipeline-using-lambda-actions-49c1fd3a3c31)
