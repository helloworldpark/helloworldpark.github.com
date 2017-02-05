---
layout: post
title:  "GPUImage에 대한 소개"
date:   2017-02-05 00:20:27 +0900
categories: jekyll update
---

## 소개글
[GPUImage 프레임워크](https://github.com/BradLarson/GPUImage)는 iOS와 macOS 기반의 GPU 렌더링 프레임워크로, Sunset Lake Software의 Brad Larson이 구현한 오픈 소스 프로젝트입니다. 이 프로젝트는 카메라 및 영상 렌더링과 관련된 작업의 효율성을 매우 높여주는데, ```OpenGL```의 구현을 밑바닥부터 해보면 더욱 더 공감이 잘 됩니다. ```OpenGL```로 뭔가를 하려고 하면 매우 귀찮습니다. ```OpenGL ES 2.0```을 기준으로 설명하면, 뭔가를 렌더링하려고 하면 ```EAGLContext```의 초기화 로직을 구현하는 것으로 시작해야 합니다. 이 작업도 매우 귀찮은 판에, 직접 짠 셰이더를 컴파일하는 작업은 물론이고, 정점의 좌표를 GPU에 넘겨주는 작업도 일일이 직접 해줘야 합니다. 프레임버퍼도 잡아줘야 하고, 텍스처 업로드 또한 일일이 신경써줘야 합니다. 게다가 모바일 환경과 같이 돌발적인 상황이 자주 발생하는 환경에서 앱이 죽지 않도록 섬세하게 신경써줘야 합니다. 이런 반복적인 작업은 지루하기만 합니다.

GPUImage의 유용함과 강력함은 이러한 반복적이고 무의미한 코드들(Context 관리, 셰이더의 컴파일, 프레임버퍼 관리 등)을 하나의 프레임워크로 만들어놓고, 개발자는 구현 로직에만 집중하게 했다는 데에 있습니다. 어떻게 보면 지향 철학은 [Spring](https://projects.spring.io/spring-framework/)과 비슷하기도 합니다만, 어쨌건 GPUImage는 ```OpenGL```과 관련된 반복적인 코드들을 개발자가 구현할 필요가 없게 해놓았습니다. 게다가 GPUImage는 이런 반복적인 코드만 구현해놓은 것이 아니라, 이미 많이 쓰이는 렌더링 기법들을 **필터**의 형태로 구현해놓았습니다. 그것도 객체지향적으로.

몇 가지 아쉬운 점들은 다음과 같습니다. 첫째, 한글화된 리소스가 거의 없습니다. 그 반대급부로 영어로 된 리소스는 대단히 많습니다만, 한국어로 문제를 해결하기는 대단히 어렵습니다. 둘째, 구현 언어가 ```Objective-C```입니다. 사실 ```Objective-C```는 우리나라에서는 ```Java```만큼 대중적인 언어는 아니고, 게다가 이제는 그 자식뻘 되는 ```Swift```에 밀려 점점 밀려나는 추세입니다. 이는 사용 가능한 플랫폼이 iOS 또는 macOS밖에 없다는 뜻이기도 합니다. 셋째, ```OpenGL```과 렌더링에 대한 기본적인 이해는 있어야 합니다. "GPUImage만 있으면 ```OpenGL```을 몰라도 카메라 앱을 만들 수 있다!" 이런 건 아닙니다. 당연히 ```OpenGL```은 어느 정도 숙지해야 합니다.

## 구조
GPUImage는 렌더링의 소스로 영상, 이미지, 카메라 등을 받을 수 있습니다. 이러한 렌더링의 소스들은 ```GPUImageOutput``` 클래스를 상속받은 클래스로 구현되어 있습니다. 렌더링의 소스는 이 클래스를 통과하면 하나의 텍스처가 되어 나가게 됩니다. 보통 이 부분은 개발자가 직접 구현할 필요가 없습니다.

개발자가 주로 구현하게 되는 부분은 이후의 렌더링 단계입니다. 각각의 렌더링 단계는 **필터**라는 단위로 이루어집니다. 마치 사진을 찍을 때 필터를 먹이듯 한다고 생각하면 됩니다. 이 각각의 필터는 ```GPUImageFilter``` 클래스를 상속받고 있는데, ```GPUImageFilter``` 클래스는 ```GPUImageOutput``` 클래스를 상속받고, ```GPUImageInput``` 프로토콜을 구현합니다. 즉, 다음과 같은 형태입니다.
{% highlight ObjC%}
@interface GPUImageFilter : GPUImageOutput <GPUImageInput>
...
{% endhighlight %}

하나의 ```GPUImageFilter는``` 입력으로 텍스처(정확하게는, 프레임버퍼에 바인딩된 텍스처)를 받고, 출력으로 렌더링된 프레임버퍼를 뱉어냅니다. 이러한 필터는 체인의 형태로 구성될 수 있어서 렌더링은 필터 체인이 끝날 때까지 연쇄적으로 이루어지게 되고, 최종 결과물은 ```GPUImageView를``` 통해 렌더링됩니다. GPUImage의 README에 나와 있는 예시를 인용하여 세피아 느낌의 효과를 주고 싶다면,
```GPUImageVideoCamera -> GPUImageSepiaFilter -> GPUImageView``` 순으로 필터 체인을 구성하면 됩니다.

## 필터의 조립
GPUImage에는 이미 많은 필터들이 구현되어 있습니다. 그 갯수도 너무 많아서 여기에 다 적는 것이 무의미하고, 또 계속 추가되고 있습니다. 그렇기 때문에 필터를 직접 구현할 필요가 없는 경우, 이미 구현된 필터를 바탕으로 조립하는 것도 가능합니다. ```GPUImageFilterGroup``` 클래스가 바로 이 역할을 하는데, 클래스의 이름 그대로 필터들의 집합을 구성할 수 있게 해줍니다. 개발자가 할 일은, 각각의 필터를 하나의 노드로 보고, 마치 Unity의 메카닉에서 애니메이션의 Finite State Machine을 구성하듯이 필터들의 관계를 정의해주면 됩니다. 즉, A 필터 다음에는 B 필터가 오고, 그 다음에는 C 필터가 오고... 이런 식으로 관계를 잘 정의해주면 됩니다.

하나의 필터가 하나의 입력만 받으라는 법은 없습니다. 때로는 두 개 이상의 필터로부터 입력을 받아야만 하는 필터도 있습니다. 예를 들어 두 필터의 결과를 받아서 Alpha Blending하는 경우에는 반드시 두 개의 필터를 받아야만 합니다. ```GPUImageFilterGroup```에서는 이렇게 여러 개의 인풋을 받는 필터 구성도 할 수 있습니다.

{% highlight ObjC%}
GPUImageFilter *passthroughFilter = [[GPUImageFilter alloc] init];
GPUImageGaussianBlurFilter *blurFilter = [[GPUImageGaussianBlurFilter alloc] init];
GPUImageDifferenceBlendFilter *differenceBlendFilter = [[GPUImageDifferenceBlendFilter alloc] init];
GPUImageAlphaBlendFilter *alphaBlendFilter = [[GPUImageAlphaBlendFilter alloc] init];
[passthroughFilter addTarget: blurFilter atTextureLocation:0];
[passthroughFilter addTarget: differenceBlendFiler atTextureLocation:0];
[blurFilter addTarget:differenceBlendFilter atTextureLocation:1];
[passthroughFilter addTarget:alphaBlendFilter atTextureLocation:0];
[differenceBlendFilter addTarget:alphaBlendFilter atTextureLocation:1];

self.initialFilters = @[passthroughFilter];
self.terminalFilter = alphaBlendFilter;
{% endhighlight %}

## 필터의 구현
필터를 직접 구현하고 싶다면, ```GPUImageFilter```를 상속받아 구현하면 됩니다. ```GPUImageFilter```의 생성자는 기본적으로 정점 셰이더와 프래그먼트 셰이더 각각 1개를 필요로 합니다. ```GPUImageFilter``` 클래스에서는 이 셰이더들을 받아서 컴파일하고 링크까지 해준 후, 삭제까지 해줍니다. 이 모든 로직을 이미 구현해놓았기 때문에, 객체 지향적인 이 프레임워크를 사용하면 셰이더를 컴파일하고 링크하는 귀찮은 작업을 또 구현할 필요가 없습니다. 개인적으로는 GPUImage의 가장 큰 장점이 이 부분이라고 생각합니다.

그렇기 때문에 개발자는 셰이더만 잘 구현하면 됩니다. 만일 ```Uniform``` 변수가 있다면, ```Uniform``` 변수에 접근할 수 있는 핸들 정도면 인스턴스 변수로 갖고 있으면 됩니다. 심지어 개발자들 고생하지 말라고 GPUImage는 ```Stringify```를 활용한 매크로 함수 ```SHADER_STRING(x)```까지 제공합니다. 게다가 렌더링과 관련된 로직들 - 텍스처를 업로드하고, Array Buffer를 바인딩하고, Draw Call을 날리는 것 등등 - 의 기본적인 구현도 다 되어있습니다(```-(void)renderToTextureWithVertices:textureCoordinates:```).

만일 더 마개조해서 쓰고 싶다면, 그것도 충분히 가능합니다. ```GLProgram``` 클래스는 위에서 언급한 컴파일된 셰이더의 래퍼 클래스라고 볼 수 있기 때문에 충분히 재사용할 수 있습니다. 또한 GPUImage는 그저 GL 함수들을 편하게 쓰기 위한 프레임워크이기 때문에, GL 함수들을 활용하여 렌더링을 직접 제어하는 것도 가능합니다. 만일 3D 렌더링을 하고 싶다면 그것도 충분히 가능하고, 필요하다면 스텐실 버퍼나 깊이 버퍼를 쓰는 것도 가능합니다(단, 이 경우에도 최종 결과물은 ```GPUImageFramebuffer```에 렌더링되어야 합니다).

이미 작성된 필터들에서 기능을 확장하고 싶을 수도 있습니다. 이런 경우라면, 원하는 필터 클래스를 확장하면 됩니다. GPUImage 자체가 대단히 객체지향적으로 잘 설계된 프로젝트라서, 아무 문제 없이 원하는 클래스를 확장할 수 있습니다. 예를 들어 입력으로 두 개 이상의 텍스처가 필요한 필터의 경우 ```GPUImageTwoInputFilter``` 클래스를 상속받으면 됩니다.

## 안드로이드에서는
안드로이드에서도 GPUImage와 유사한 오픈소스 프로젝트가 있습니다. [GPUImage for Android](https://github.com/CyberAgent/android-gpuimage)라는 프로젝트는 GPUImage의 안드로이드 포팅 버전인데, 필터 방식의 렌더링이라는 점에서 기본적인 아이디어가 같습니다.

