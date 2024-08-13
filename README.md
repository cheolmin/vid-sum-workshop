# 01. Prerequisites

## Lambda Layer&#x20;



Lambda Layer는 추가 코드 또는 데이터를 포함하는 .zip 파일 아카이브입니다. Layer에는 일반적으로 라이브러리 종속 항목, 사용자 지정 런타임 또는 구성파일이 포함됩니다.



Layer 사용을 고려하는 데에는 여러가지 이유가 있습니다.

* 배포 패키지의 크기를 줄이기 위해. 모든 함수 종속 항목을 함수 코드와 함께 배포 패키지에 포함하는 대신 계층에 배치합니다. 이렇게 하면 배포 패키지가 작고 체계적으로 유지됩니다.
* 핵심 함수 로직을 종속 항목과 분리하기 위해. 계층을 사용하면 함수 코드와 독립적으로 함수 종속 항목을 업데이트할 수 있으며 그 반대의 경우도 마찬가지입니다. 이렇게 하면 관심사를 분리하고 함수 로직에 집중할 수 있습니다.
* 여러 함수에서 종속 항목을 공유하기 위해. 계층을 생성한 후 계정의 여러 함수에 적용할 수 있습니다. 계층이 없으면 각 개별 배포 패키지에 동일한 종속 항목을 포함해야 합니다.
* Lambda 콘솔 코드 편집기를 사용하기 위해. 코드 편집기는 함수 코드의 부분 업데이트를 빠르게 테스트하는 데 유용한 도구입니다. 그러나 배포 패키지 크기가 너무 큰 경우 편집기를 사용할 수 없습니다. 계층을 사용하면 패키지 크기가 줄어들고 코드 편집기를 사용할 수 있습니다.

다음 다이어그램에서는 종속 항목을 공유하는 두 함수 간의 중요 아키텍처 차이를  보여줍니다. 하나는 Lambda 계층을 사용하고 다른 하나는 사용하지 않습니다.



<figure><img src=".gitbook/assets/image (51).png" alt=""><figcaption></figcaption></figure>

이번 워크샵에서는 Layer 생성 및 삭제와 관련된 내용은 다루지 않습니다. 해당 내용은 아래 링크에서 확인하실 수 있습니다.

{% embed url="https://docs.aws.amazon.com/ko_kr/lambda/latest/dg/chapter-layers.html" %}

## Lambda Layer 관련 자료

#### FFMpeg Layer

{% file src=".gitbook/assets/ffmpeg.zip" %}

#### OpenCV Layer

{% file src=".gitbook/assets/opencv-python-headless-4.6.0.66-final.zip" %}

#### Requests Layer

arn:aws:lambda:us-east-1:770693421928:layer:Klayers-p39-requests:19



## Lambda Layer 추가

1. S3에 위 첨부파일을 업로드합니다.
2. AWS Console 에서 Lambda를 검색합니다.

<figure><img src=".gitbook/assets/image (52).png" alt=""><figcaption></figcaption></figure>

2. Lambda 페이지 내 메뉴에서 Layer를 클릭하고, 우측 상단의 함수 생성 버튼을 클릭합니다.

<figure><img src=".gitbook/assets/image (54).png" alt=""><figcaption></figcaption></figure>

3. Layer의 이름을 입력하고 호환 아키텍처 및 런타임을 설정한 후 생성을 클릭합니다.

> **위 첨부 파일의 호환 아키텍처는 x86\_64 / 호환 런타임은 Python 3.9 입니다.**

<figure><img src=".gitbook/assets/image (55).png" alt=""><figcaption></figcaption></figure>

4. FFMpeg / OpenCV Layer 모두 생성합니다.

<figure><img src=".gitbook/assets/image (57).png" alt=""><figcaption></figcaption></figure>

## 버킷 생성



1. 솔루션에서 작업할 버킷을 생성합니다. S3 콘솔에서 적절한 이름의 버킷을 생성합니다.

<figure><img src=".gitbook/assets/image (69).png" alt=""><figcaption></figcaption></figure>

2. 나머지 설정은 기본으로 세팅한 후 생성을 클릭합니다.
3. 버킷 내에 작업물이 저장될 폴더를 생성합니다.

```
videos : input video가 저장될 경로
audios : audio file이 저장될 경로
transcribe : STT 결과가 저장될 경로
result : 최종 subclip이 저장될 경로
```



## 테스트용 비디오 파일

{% file src=".gitbook/assets/test.mp4" %}
