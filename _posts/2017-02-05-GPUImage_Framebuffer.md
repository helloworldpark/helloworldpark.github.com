---
layout: post
title:  "GPUImage의 프레임버퍼 관리"
date:   2017-02-05 14:20:27 +0900
categories: jekyll update
---

이 글은 OpenGL ES 2.0을 기준으로 설명하며, 이에 대한 어느 정도의 이해를 가정합니다.

## 프레임버퍼란
프레임버퍼란 쉽게 말해 연습용 도화지 같은 것입니다. 그림을 그릴 때 한 번에 캔버스에 그려버릴 수도 있지만, 그게 도저히 불가능할 수도 있습니다. 그럴 때에는 연습장에다가 그리고 나서 옮겨 그리는 전략을 취할 수 있는데, 그럴 때 쓰는 연습장같은 게 프레임버퍼입니다.

프레임버퍼 자체는 사실 버퍼들의 참조를 들고 있을 뿐이고, 진짜 알맹이는 프레임버퍼에 붙어있는 Attachment입니다. Renderbuffer로 쓰이는 0번 프레임버퍼를 제외하면, Color Attachment, Depth Attachment, Stencil Attachment를 붙일 수 있습니다.

## GPUImage의 프레임버퍼
GPUImage에서는 프레임버퍼의 생성과 바인딩 등의 로직을 ```GPUImageFramebuffer``` 클래스로 구현해두었습니다. 그런데 GPUImage의 특성상 2D 렌더링이 주가 될 수밖에 없어서, ```GPUImageFramebuffer는``` Color Attachment만 붙어있습니다.

### 프로퍼티
프로퍼티로 다음과 같은 것들이 있습니다.
{% highlight ObjC %}
@property(readonly) CGSize size;
@property(readonly) GPUTextureOptions textureOptions;
@property(readonly) GLuint texture;
@property(readonly) BOOL missingFramebuffer;
{% endhighlight %}

프로퍼티들은 전부 읽기 전용입니다. 보면 상식적으로 있어야 할 것밖에 없는데, 그 중에 ```missingFramebuffer```라는 녀석이 특이합니다. 이 프로퍼티를 보고 ```GPUImageFramebuffer``` 객체를 프레임버퍼로 쓰지 않아도 되는 것 아니냐 하고 생각할 수 있는데, 맞습니다. 굳이 프레임버퍼로 쓰는 것이 아니라 텍스처를 들고 있는 객체로 쓸 수도 있습니다.

### 생성자
생성자는 다음과 같은 것들이 제공됩니다.
{% highlight ObjC %}
- (id)initWithSize:(CGSize)framebufferSize;
- (id)initWithSize:(CGSize)framebufferSize textureOptions:(GPUTextureOptions)fboTextureOptions onlyTexture:(BOOL)onlyGenerateTexture;
- (id)initWithSize:(CGSize)framebufferSize overriddenTexture:(GLuint)inputTexture;
{% endhighlight %}

일단 사이즈는 기본으로 제공되어야 하고, 두 번째와 세 번째를 보면 옵션을 줄 수 있습니다. 두 번째 생성자에서 ```textureOptions```를 줄 수 있는데, ```GPUTextureOptions``` 구조체를 만든 후에 필요한 인자를 잘 채워넣어주면 됩니다. ```onlyTexture``` 인자는 ```GPUImageFramebuffer``` 객체를 텍스처 셔틀로 쓸 것인지 정해줍니다(이렇게 되면 프레임버퍼의 핸들은 0이 됩니다). 보통은 ```NO```를 주게 됩니다.

세 번째 생성자를 보면 ```overridenTexture```라는 인자가 있는데, 이 인자는 기존에 이미 텍스처를 만들어둔 경우에 활용할 수 있습니다. 즉, 이 프레임버퍼 객체에 기존에 만들어둔 텍스처를 연결하는 것입니다.

그런데 사실 이 생성자들을 개발자가 직접 **호출할 일은 많지 않습니다**. 그 이유는 밑에서 설명하겠습니다.

### 사용
```GPUImageFramebuffer``` 객체를 사용하려면 다음 메소드를 호출합니다.
{% highlight ObjC %}
- (void)activateFramebuffer;
{% endhighlight %}

이 메소드를 호출하게 되면 이 프레임버퍼를 GPU에 바인딩합니다.

### 이미지 캡처
이미지를 캡처할 수 있는 메소드도 있습니다.
{% highlight ObjC %}
- (CGImageRef)newCGImageFromFramebufferContents;
- (void)restoreRenderTarget;
{% endhighlight %}

```newCGImageFromFramebufferContents```는 프레임버퍼에서 이미지를 읽은 후에 CGImageRef로 만들어서 반환합니다. 

### 로우레벨 접근
```GPUImageFramebuffer는``` 로우레벨의 바이트 접근도 제한적으로나마 허용합니다. 영상 처리를 위해서는 프레임버퍼의 픽셀 정보를 읽어와야 할 일이 있는데, 그럴 때 아래의 메소드들을 사용하면 편리하게 읽어올 수 있습니다.
{% highlight ObjC %}
- (void)lockForReading;
- (void)unlockAfterReading;
- (NSUInteger)bytesPerRow;
- (GLubyte *)byteBuffer;
{% endhighlight %}

픽셀 정보를 읽을 때 ```GPUImageFramebuffer는``` 내부의 ivar인 ```renderTarget```을 사용하는데, ```renderTarget```의 타입은 ```CVPixelBufferRef```입니다. ```CVPixelBufferRef``` 변수로 읽기나 쓰기 작업을 하려면 Lock을 걸어줘야 하는데, 그 작업을 해주는 메소드가 ```lockForReading```, ```unlockForReading```입니다.

한 가지 주의할 점은, 이 메소드는 **iOS에서만 실행됩니다**. macOS에서는 메소드를 실행해도 아무 일도 일어나지 않거나 ```NULL```이 반환됩니다. 또한 이 메소드는 너무 자주 호출하면 앱의 퍼포먼스에 심각한 영향을 미칠 수 있습니다.


## GPUImage의 프레임버퍼 관리
사실 이 부분이 ```GPUImageFramebuffer``` 사용에서 가장 중요하고 특징적인 부분입니다. ```GPUImageFramebuffer는``` 자체적으로 레퍼런스 카운팅을 사용합니다. ARC와는 별개로 돌아가는 부분인데, 이 부분을 이해하려면 GPUImage에서 프레임버퍼를 어떻게 관리하는지 알아야 합니다.

GPUImage에서는 ```GPUImageFramebuffer```를 재활용합니다. 예를 들어, 총 5개의 필터가 서로 연결되어 있다고 하겠습니다.
{% highlight ObjC %}
A -> B -> C -> D -> E
{% endhighlight %}

이렇게 써놓고 보면 프레임버퍼 객체는 총 5개가 있어야 할 것 같습니다. 그런데 이건 사실 낭비입니다. 왜냐하면 ```A```를 렌더링한 결과는 ```B```의 입력이 되지, ```C```의 입력이 되지는 않거든요. 만일 ```A```와 ```B```를 렌더링하고 나서, ```A```에서 썼던 프레임버퍼를 ```C```에서 재활용할 수 있다면 5개가 아니라 **2개만 갖고** 저 필터 체인을 전부 처리할 수 있을 것입니다.

GPUImage는 이러한 재활용 로직을 구현해놓았는데, 그 역할을 하는 것이 ```GPUImageFramebufferCache```입니다. 실제로, ```GPUImageFilter``` 클래스의 ```-(void)renderToTextureWithVertices:textureCoordinates:``` 메소드를 보면, 프레임버퍼를 직접 만드는 것이 아니라 ```GPUImageFramebufferCache```를 통해 생성하고 있습니다.
{% highlight ObjC %}
outputFramebuffer = [[GPUImageContext sharedFramebufferCache] fetchFramebufferForSize:[self sizeOfFBO] textureOptions:self.outputTextureOptions onlyTexture:NO];
{% endhighlight %}

그렇기 때문에, 사실 개발자는 ```GPUImageFramebuffer```의 생성자를 직접 호출할 일이 거의 없습니다(물론 최적화 등을 위해서 개발자가 직접 호출할 수는 있겠습니다). ```GPUImageFramebufferCache```를 통해 이미 생성된 프레임버퍼가 있으면 그 프레임버퍼를 재활용하고, 만일 현재 프레임버퍼 캐시에 남아있는 ```GPUImageFramebuffer``` 객체가 없다면 ```GPUImageFramebufferCache```가 생성해서 돌려줍니다. 게다가 ```GPUImageFramebufferCache```도 개발자가 직접 생성할 일이 없습니다. GPUImageContext에 이미 ```sharedFramebufferCache```라는 공유 캐시가 있기 때문입니다.

이제 레퍼런스 카운팅과 관련된 함수들을 보겠습니다.
{% highlight ObjC %}
- (void)lock;
- (void)unlock;
- (void)clearAllLocks;
- (void)disableReferenceCounting;
- (void)enableReferenceCounting;
{% endhighlight %}

```lock```은 프레임버퍼의 레퍼런스 카운트를 하나 올립니다. 여기에서 레퍼런스 카운트는 ```GPUImageFramebuffer```의 ivar 중 하나입니다. 반대로 ```unlock```은 레퍼런스 카운트를 하나 낮춥니다. 그리고 ```clearAllLocks```는 레퍼런스 카운트를 0으로 초기화합니다. 만일 레퍼런스 카운트가 0이 되면 ```GPUImageFramebuffer```객체는 스스로 ```GPUImageFramebufferCache```로 돌아갑니다. 바로 이런 식으로 GPUImage에서 프레임버퍼를 관리합니다.

```disableReferenceCounting```과 ```enableReferenceCounting```은 그저 레퍼런스 카운팅을 사용할 것인지 아닌 것인지를 설정하는 함수입니다. 기본값은 ```YES```이지만, 만일 ```NO```로 하겠다면 개발자가 직접 ```GPUImageFramebuffer```의 생명주기를 관리해줘야 합니다.

한 가지 주의할 점은, ```lock```과 ```unlock```은 **반드시 짝이 맞아야** 합니다. 만일 ```lock```만 계속 걸게 되면 그 ```GPUImageFramebuffer``` 객체는 캐시로 회수되지 않기 때문에 메모리 누수가 생기게 됩니다. 반대로 ```unlock```을 너무 많이 호출하면 ```NSAssert```에서 실패가 나고, 앱이 크래시가 나게 됩니다.
