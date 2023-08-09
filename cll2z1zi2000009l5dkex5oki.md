---
title: "메서드 호출을 인터셉터로 대체하기"
datePublished: Wed Aug 09 2023 00:08:06 GMT+0000 (Coordinated Universal Time)
cuid: cll2z1zi2000009l5dkex5oki
slug: exploring-the-dotnet-8-preview-changing-method-calls-with-interceptors
canonical: https://andrewlock.net/exploring-the-dotnet-8-preview-changing-method-calls-with-interceptors/
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1691536740662/2c5d6b53-5273-4402-83b2-85b73df46c23.jpeg
tags: net, dotnet

---

> Andrew Lock님의 [Replacing method calls with Interceptors](https://andrewlock.net/exploring-the-dotnet-8-preview-changing-method-calls-with-interceptors/)을 DeepL을 도움으로 번역하였습니다.

이 글은 [.NET 8 미리 보기 살펴보기 시리즈](https://dimohy.hashnode.dev/series/exploring-dotnet8-preview)의 다섯 번째 글입니다.

[1부 - 새로운 구성 바인더 소스 생성기 사용하기](https://dimohy.hashnode.dev/7ioi66gc7jq0ioq1royessdrsjtsnbjrjzqg7iam7iqkioydneyeseq4scdsgqzsmqk)  
[2부 - 미니멀 API AOT 컴파일 템플릿](https://dimohy.hashnode.dev/the-minimal-api-aot-compilation-template)  
[3부 - WebApplication.CreateBuilder()와 새로운 CreateSlimBuilder() 메서드 비교하기](https://dimohy.hashnode.dev/webapplicationcreatebuilder-createslimbuilder)  
[4부 - 새로운 미니멀 API 소스 생성기 살펴보기](https://dimohy.hashnode.dev/exploring-the-dotnet-8-preview-exploring-the-new-minimal-api-source-generator)  
5부 - 메서드 호출을 인터셉터로 대체하기(이 게시물)

이 시리즈에서는 .NET 8 프리뷰에 포함된 몇 가지 새로운 기능을 살펴봅니다. 이 글에서는 C#12 미리보기 기능인 [인터셉터](https://devblogs.microsoft.com/dotnet/new-csharp-12-preview-features/#interceptors)를 살펴보고, 작동 방식과 유용한 이유를 살펴보고, [이전 글](https://dimohy.slogs.dev/exploring-the-dotnet-8-preview-exploring-the-new-minimal-api-source-generator)에서 소개한 미니멀 API 소스 생성기가 어떻게 업데이트되었는지 설명합니다.

> 이 게시물은 모두 프리뷰 빌드를 사용하고 있으므로 2023년 11월에 .NET 8이 최종 출시되기 전에 일부 기능이 변경(또는 제거)될 수 있습니다!

## 인터셉터란 무엇이며 왜 필요한가요?

인터셉터는 애플리케이션의 메서드 호출을 대체 메서드로 대체(또는 "인터셉트")할 수 있는 C#12의 흥미로운 새 실험적 기능입니다. 앱이 컴파일되면 컴파일러가 자동으로 원래 메서드 호출을 대체 메서드로 "교체"합니다.

여기서 당연한 질문은 왜 그렇게 하려고 하느냐는 것입니다. 대체 메서드를 직접 호출하지 않는 이유는 무엇일까요?

주된 이유는 [이 시리즈에서 여러 차례 설명한 바 있는](https://dimohy.hashnode.dev/the-minimal-api-aot-compilation-template) 미리 컴파일(AOT) 때문입니다(이 기능이 .NET 8의 핵심 기능이라는 점을 감안하면 당연한 일입니다). 인터셉터는 특별히 AOT를 위한 것은 아니지만, 분명히 AOT를 염두에 두고 설계되었습니다. 인터셉터를 사용하면 이전에는 AOT에 적합하지 않았던 코드를 가져와서 소스 생성 버전으로 바꿀 수 있습니다.

고객은 코드를 변경할 필요가 없으며, 소스 생성기가 메서드 호출을 자동으로 "업그레이드"하여 소스 생성 버전을 사용하기만 하면 됩니다. 익숙하게 들린다면 [구성 소스 생성기](https://andrewlock.net/exploring-the-dotnet-8-preview-using-the-new-configuration-binder-source-generator/)와 [기존의 미니멀 API 소스 생성기](https://andrewlock.net/exploring-the-dotnet-8-preview-exploring-the-new-minimal-api-source-generator/)가 이미 그렇게 하고 있기 때문입니다! 차이점은 이러한 생성기는 현재 메서드 특이성 규칙에 의존하여 컴파일러가 소스 생성된 메서드를 사용하도록 "속이는" 것입니다. 인터셉터는 대체를 명시적으로 만듭니다.

이론적으로 인터셉터를 사용하면 구성 소스 생성기나 미니멀 API 소스 생성기와 같은 소스 생성기를 더 쉽게 빌드할 수 있습니다. 심지어 [정규식 생성기](https://learn.microsoft.com/en-us/dotnet/standard/base-types/regular-expression-source-generators)나 [로깅 생성기](https://andrewlock.net/exploring-dotnet-6-part-8-improving-logging-performance-with-source-generators/)와 같은 기존 소스 생성기를 개선할 수도 있습니다. 생성기가 요구하는 패턴을 사용하기 위해 코드를 변경할 필요 없이 생성기가 자동으로 기존 코드를 개선할 수 있습니다! 적어도 이론적으로는 말이죠.

> 몇 달 전에 [.NET 커뮤니티 스탠드업에서 인터셉터에 대한 훌륭한 토론](https://www.youtube.com/watch?v=X1_QeH1yAto&list=PLdo4fOcmZ0oX-DBuRG4u58ZTAJgBAeQ-t&index=12)이 있었습니다. 꼭 시청해 보시길 강력히 추천합니다!

이 모든 것이 매우 추상적이므로 다음 섹션에서는 인터셉터의 작동 원리를 이해할 수 있도록 매우 간단한 예시를 보여드리겠습니다.

## 간단한 (비실용적인) 작동 인터셉터 예시

이 섹션에서는 인터셉터의 매우 비실용적인 예를 보여드리겠습니다. 인터셉터는 소스 생성기와 함께 사용하도록 설계되었지만 반드시 그럴 필요는 없으므로 간단하게 "원시" 인터셉터만 작성해 보겠습니다!

우선, .NET 8 미리보기 6 이상을 사용해야 합니다. 예를 들어 다음을 실행하여 새 앱을 만듭니다([새 AOT 템플릿](https://andrewlock.net/exploring-the-dotnet-8-preview-the-minimal-api-aot-template/)일 필요는 없음)

```bash
dotnet new web
```

이렇게 하면 다음과 같은 앱이 생성됩니다.

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.Run();
```

다음으로, C#12 미리보기 기능을 사용하도록 .csproj 파일을 업데이트해야 합니다. 또한 실험적인 인터셉터 기능을 명시적으로 활성화해야 합니다.

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <!-- 👇 Add this to enable C#12 preview features -->
    <LangVersion>preview</LangVersion>
    <!-- 👇 Enable the interceptors feature -->
    <Features>InterceptorsPreview</Features>
  </PropertyGroup>

</Project>
```

다음으로 프로젝트에서 `[InterceptsLocation]` 특성을 정의해야 합니다. 이 특성은 인터셉터 동작을 구동하지만 아직 기본 클래스 라이브러리(BCL)에 포함되어 있지 않으므로 직접 정의해야 합니다. 소스 생성기에서 인터셉터를 사용하는 경우에도 마찬가지입니다.

프로젝트의 어딘가에 다음과 같이 속성을 정의하세요.

```csharp
namespace System.Runtime.CompilerServices
{
    [AttributeUsage(AttributeTargets.Method, AllowMultiple = true)]
    sealed class InterceptsLocationAttribute(string filePath, int line, int column) : Attribute
    {
    }
}
```

> 이 정의를 소스 생성기와 동일한 파일에 포함하면 [파일 로컬 범위](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-11#file-local-types)를 사용하여 '누수'를 방지할 수 있습니다.

마지막으로 인터셉터를 만들 수 있습니다. 이 데모에서는 `MapGet()` 호출을 가로채서 경로 패턴을 콘솔에 기록한 다음 원래 메서드를 호출하는 메서드를 만들어 보겠습니다. 이것은 그다지 유용하지 않습니다. 그냥 데모일 뿐입니다 😅.

인터셉터를 만들려면 다음이 필요합니다.

* 정적 클래스를 생성합니다.
    
* 대상 메서드와 정확히 동일한 매개변수와 반환 유형을 가진 메서드를 정의합니다.
    
* 인터셉터에 `[InterceptsLocation]` 특성을 추가하여 인터셉트하려는 메서드의 소스 위치를 지정합니다.
    

이 예제에서는 `MapGet` 확장 메서드를 가로채고 싶으므로 [이와 동일한 시그니처](https://github.com/dotnet/aspnetcore/blob/a56e968c19be1275db6dc462310d723615b006a7/src/Http/Routing/src/Builder/EndpointRouteBuilderExtensions.cs#L230)를 가진 메서드를 만들어야 합니다.

```csharp
public static RouteHandlerBuilder MapGet(
        this IEndpointRouteBuilder endpoints,
        [StringSyntax("Route")] string pattern,
        Delegate handler);
```

이는 충분히 쉬우며, 까다로운 부분은 인터셉트하려는 메서드의 첫 번째 문자의 위치로 `[InterceptsLocation]`을 설정하는 것입니다!

> 이 끔찍한 해킹 예제에서는 까다로울 뿐입니다. 소스 생성기에서는 소스 위치에 쉽게 액세스할 수 있으므로 문제가 없습니다.

이 모든 것을 종합하면 인터셉터의 모습은 다음과 같습니다.

```csharp
static class Interception
{
    // 👇 Define the file, line, and column of the method to intercept
    [InterceptsLocation(@"C:\testapp\Program.cs", line: 4, column: 5)]
    public static RouteHandlerBuilder InterceptMapGet( // 👈 The interceptor must
        this IEndpointRouteBuilder endpoints,          // have the same signature
        string pattern,                                // as the method being
        Delegate handler)                              // intercepted
    {
        Console.WriteLine($"Intercepted '{pattern}'" );

        return endpoints.MapGet(pattern, handler);
    }
}
```

그게 다입니다. 앱을 실행하면 인터셉터가 작동하는 것을 확인할 수 있습니다!

```bash
Intercepted '/'
...
```

엔드포인트가 여러 개인 경우 어떻게 하나요?

```csharp
app.MapGet("/", () => "Hello World!");
app.MapPost("/{name}", (string name) => "Hello {name}!");
```

다른 HTTP 동사이긴 하지만 매개변수와 반환 유형이 동일하기 때문에 기존 인터셉터를 재사용할 수 있습니다.

```csharp
static class Interception
{
    // 👇 Multiple attributes are fine
    [InterceptsLocation(@"C:\testapp\Program.cs", line: 4, column: 5)]
    [InterceptsLocation(@"C:\testapp\Program.cs", line: 5, column: 5)]
    public static RouteHandlerBuilder InterceptMapGet(
        this IEndpointRouteBuilder endpoints,
        string pattern,
        Delegate handler)
    {
        Console.WriteLine($"Intercepted '{pattern}'" );

        return endpoints.MapGet(pattern, handler);
    }
}
```

메서드 오버로드에 주의하세요. 기존 인터셉터로 다음 메서드 호출을 가로채려고 하면

```csharp
app.MapGet("/ping", (HttpContext context) => Task.CompletedTask);
```

컴파일 타임 예외가 발생합니다.

```bash
error CS9144: Cannot intercept method 'EndpointRouteBuilderExtensions.MapPost
(IEndpointRouteBuilder, string, RequestDelegate)' with interceptor 'Interception.
InterceptMapGet(IEndpointRouteBuilder, string, Delegate)' because the signatures do not match.
```

오류 메시지에서 볼 수 있듯이, [인터셉터와 매개변수 유형이 다른 메서드](https://github.com/dotnet/aspnetcore/blob/a56e968c19be1275db6dc462310d723615b006a7/src/Http/Routing/src/Builder/EndpointRouteBuilderExtensions.cs#L65), 즉 `Delegate`가 아닌 `RequestDelegate`를 인터셉트하려고 합니다. 다시 말하지만, 소스 제너레이터의 경우 제너레이터에서 매개변수 유형을 식별하고 메서드를 무시하거나 적절한 인터셉터를 출력할 수 있으므로 큰 문제가 되지 않습니다.

이제 장난감 예제를 이해하셨으니 실제 소스 제너레이터가 인터셉터를 어떻게 사용하는지 살펴보겠습니다!

## 인터셉터 사용을 위한 미니멀 API 생성기 업데이트

이전 게시물에서 .NET 8 미리보기 6에서 제공되는 [미니멀 API 소스 생성기](https://dimohy.hashnode.dev/exploring-the-dotnet-8-preview-exploring-the-new-minimal-api-source-generator)에 대해 설명했습니다. 하지만 해당 게시물을 게시하기 직전에 소스 생성기 구현을 인터셉터를 사용하도록 대체하는 [PR이 병합](https://github.com/dotnet/aspnetcore/pull/48817)되었습니다!

> 당시 저는 이 사실에 조금 놀랐습니다. 인터셉터가 [.NET 8에서 실험 단계에 머물러 있다는 점](https://devblogs.microsoft.com/dotnet/new-csharp-12-preview-features/#interceptors)을 감안할 때, 저는 소스 생성기(실험 단계는 아니지만 완전히 지원될 예정)에서 해당 기능을 사용할 수 없을 것이라고 생각했습니다. 하지만 [Safia Abdalla가 .NET 8에서 미니멀 API 생성기가 인터셉터를 사용하는 방법(그리고 그 이유)](https://twitter.com/captainsafia/status/1682096427881889792)에 대한 스레드로 응답해 주었습니다!

이 글을 쓰는 시점에 새 소스 생성기는 아직 출시되지 않았으므로(.NET 8 프리뷰 7에 출시될 예정임) 이 섹션의 세부 사항은 매우 잠정적인 것입니다. 변경 사항에 대한 설명은 유닛 테스트에 사용된 스냅샷을 기반으로 하고 있습니다.

> 스냅샷은 이 시나리오에 완벽하기 때문에 ASP.NET Core 팀에서 소스 생성기 테스트에 스냅샷 테스트를 사용하고 있는 것을 보고 반가웠습니다. [이전 게시물에서 자체 생성기에서 스냅샷 테스트를 사용하는 방법을 설명했지만](https://andrewlock.net/creating-a-source-generator-part-2-testing-an-incremental-generator-with-snapshot-testing/), 저는 [Simon Cropp](https://twitter.com/SimonCropp)의 훌륭한 [Verify](https://github.com/VerifyTests/Verify) 라이브러리를 사용했지만 Microsoft는 훨씬 더 간단한(더 조잡한) 대안을 사용하고 있는 것 같습니다.

[이전 글](https://dimohy.slogs.dev/exploring-the-dotnet-8-preview-exploring-the-new-minimal-api-source-generator)에서 프리뷰 6에서 생성된 코드에 대해 자세히 설명했지만 인터셉터를 사용하면 어떻게 달라지는지 확인할 수 있도록 간단히 요약해 보겠습니다.

### 인터셉터가 없는 프리뷰 6의 미니멀 API 생성기

다시 말해, 인터셉터 없이 [미니멀 API 소스 생성기는 생성된 메서드가 "선택"되었는지 확인하기 위해 메서드 특이성에 의존](https://andrewlock.net/exploring-the-dotnet-8-preview-exploring-the-new-minimal-api-source-generator/#intercepting-method-calls-using-source-generators)하며, `[CallerFilePath]` 및 `[CallerLineNumber]` 인수를 사용하여 소스 코드에서 위치를 캡처합니다. 그런 다음 이러한 호출자 정보 인수는 사전의 인덱스 키로 사용되어 최종적으로 생성된 엔드포인트 핸들러 정의가 포함된 [`RequestDelegateFunc`를 반환](https://andrewlock.net/exploring-the-dotnet-8-preview-exploring-the-new-minimal-api-source-generator/#looking-at-the-generated-requestdelegatefunc)합니다.

임시 '가로채기' 인프라에는 꽤 많은 메서드와 유형이 있습니다:

1. "특정성을 통한 가로채기" 메서드는 앱의 각 `MapGet()`/`MapPost()` 등의 호출에 대해 정의됩니다. 여기에는 핸들러의 시그니처와 일치하는 (덜 특정적인 `Delegate` 대신) `Func<>`가 사용됩니다. 서명이 다를 때마다 이 메서드의 버전이 다르므로 `Func<string>`과 `Func<string, string>`에는 서로 다른 메서드가 필요합니다.
    

```csharp
internal static RouteHandlerBuilder MapGet(
    this IEndpointRouteBuilder endpoints,
    string pattern,
    Func<string> handler,  // 👈 matches the endpoint handler definition signature
    [CallerFilePath] string filePath = "",
    [CallerLineNumber]int lineNumber = 0);
```

1. 이 메서드는 `MapCore()`를 호출하여 모든 세부 정보를 전달합니다. `MapCore()`는 제공된 `filePath`와 `lineNumber`를 딕셔너리의 키로 사용합니다.
    

```csharp
file static class GeneratedRouteBuilderExtensionsCore
{
    internal static RouteHandlerBuilder MapCore(
        this IEndpointRouteBuilder routes,
        string pattern,
        Delegate handler,
        IEnumerable<string>? httpMethods,
        string filePath,
        int lineNumber)
    {
        // 👇Use the filePath and lineNumber as an index into the map dictionary
        var (populateMetadata, createRequestDelegate) = map[(filePath, lineNumber)];

        // Pass the functions to the minimal API internals
        return RouteHandlerServices.Map(routes, pattern, handler, httpMethods, populateMetadata, createRequestDelegate);
    }
}
```

1. 딕셔너리에는 앱의 모든 `MapGet()` 호출에 대한 항목이 있습니다. 항목의 값은 필수 `MetadataPopulator`와 `RequestDelegateFunc` 구현을 포함하는 튜플입니다.
    

```csharp
file static class GeneratedRouteBuilderExtensionsCore
{
    private static readonly Dictionary<(string, int), (MetadataPopulator, RequestDelegateFactoryFunc)> map = new()
    {
        [(@"C:\testapp\Program.cs", 4)] = // "/", () => "Hello World!"
        (
            (methodInfo, options) => { /*   */ }, // MetadataPopulator
            (del, options, inferredMetadataResult) => { /*   */ }, // RequestDelegateFactoryFunc
        ),
        [(@"C:\testapp\Program.cs", 5)] = // "/{name}", (string name) => "Hello {name}!"
        (
            (methodInfo, options) => { /*   */ }, // MetadataPopulator
            (del, options, inferredMetadataResult) => { /*   */ }, // RequestDelegateFactoryFunc
        ),
    }
}
```

이제 인터셉터를 사용하면 상황이 어떻게 달라지는지 살펴봅시다!

### 인터셉터 기반 미니멀 API 생성기

아래 표시된 코드는 [여기](https://github.com/dotnet/aspnetcore/blob/1121b2bbb123ad9044d593955e7a2ef863fcf2e5/src/Http/Http.Extensions/test/RequestDelegateGenerator/RequestDelegateCreationTests.Responses.cs#L26) 유닛 테스트의 [스냅샷 결과](https://github.com/dotnet/aspnetcore/blob/1121b2bbb123ad9044d593955e7a2ef863fcf2e5/src/Http/Http.Extensions/test/RequestDelegateGenerator/Baselines/Multiple_MapAction_NoParam_StringReturn.generated.txt)를 기반으로 합니다.

> 이 코드는 아직 프리뷰 버전이 아니므로 11월 정식 출시 전에 변경될 수 있다는 점을 기억하세요!

이전과 마찬가지로 구현 간에 변경되지 않으므로 `MetadataPopulator` 및 `RequestDelegateFactoryFunc` 정의는 숨기겠습니다. 가장 큰 변화는 인터셉터 기반 소스 생성기에 3단계 대신 1단계만 필요하다는 것입니다.

```csharp
file static class GeneratedRouteBuilderExtensionsCore
{
    private static readonly string[] GetVerb = new[] { global::Microsoft.AspNetCore.Http.HttpMethods.Get };

    [InterceptsLocation(@"C:\testapp\Program.cs", 4, 5)]
    internal static RouteHandlerBuilder MapGet0(
        this IEndpointRouteBuilder endpoints,
        string pattern,
        Delegate handler)
    {
        MetadataPopulator populateMetadata = (methodInfo, options) => { /*   */ },
        RequestDelegateFactoryFunc createRequestDelegate = (del, options, inferredMetadataResult) => { /*   */ },

        return MapCore(endpoints, pattern, handler, GetVerb, populateMetadata, createRequestDelegate);
    }
    // ...
}
```

그게 다입니다! (잠재적으로) 취약한 "특이성" 트릭이나 `[CallerFilePath]` 및 사전 조회가 필요하지 않습니다. 인터셉터 속성을 사용하면 각 `MapGet()`/`MapPost()` 호출을 생성된 메서드에 깔끔하게 연결할 수 있습니다. 각기 다른 엔드포인트 핸들러 시그니처에는 각기 다른 `RequestDelegateFunc`가 필요합니다. 여러 엔드포인트에 동일한 서명이 있는 경우, 정의에 `[InterceptsLocation]` 속성을 추가하여 생성된 정의를 공유할 수 있습니다.

인터셉터 접근 방식에는 몇 가지 이점이 있습니다.

* 더 빨라짐(단일 메서드 호출 및 사전 조회 없음)
    
* 이해하기 쉬움(이 코드는 프로젝트에서 생성되며, 생성기 소비자로서 더 쉽게 따라할 수 있음)
    
* 유지 관리가 더 쉬움(이해하기 쉬우므로)
    
* 메서드 선택 관련 문제 제거(소스에서 생성된 버전이 선택된다는 보장이 없으므로 기술적으로 더 구체적인 메서드를 만들 수 있습니다.)
    

이 구현의 유일한 단점은 인터셉터를 사용한다는 점인데, 이는 실험적인 기능이며 11월에 출시되는 .NET 8 릴리스에서도 그대로 유지될 예정입니다. 즉, 미니멀 API AOT 소스 생성기가 (현재) 프로젝트에서 인터셉터 기능을 자동으로 활성화합니다. 이로 인해 문제가 발생하지는 않겠지만, 누군가는 이에 대해 짜증을 낼 것 같습니다 😅.

## 인터셉터에 대한 제한 사항

이 글에서는 인터셉터를 사용하는 일반적인 방법과 이를 사용하는 실제 소스 생성기 구현의 예, 그리고 소스 생성기 작성자의 작업을 더 쉽게 만들어주는 방법에 대해 살펴봤습니다.

하지만 현재 인터셉터 설계에 대한 비판은 [dotnet/csharplang#7009](https://github.com/dotnet/csharplang/issues/7009) 이슈에서 꽤 많이 제기되었습니다. 100개 이상의 댓글이 달렸으니 토끼굴에 들어가고 싶으시다면 즐겨보세요! 여러분의 기대치를 낮추기 위해 아래에 몇 가지 명백한 디자인 한계를 짚어보았습니다!

### 1\. 메서드만 해당

현재 인터셉터 설계는 메서드 인터셉트에만 초점을 맞추고 있습니다. 즉, 속성 접근자나 더 중요한 생성자는 가로챌 수 없습니다. 생성자 지점은 흥미로운데, 사용자가 코드를 변경할 필요 없이 소스에서 생성된 `Regex` 구현으로 `new Regex("a-z+")`과 같은 것을 자동으로 가로챌 수 있다는 의미이기 때문입니다. 현재 `Regex` 또는 `ILogger` 소스 생성기를 사용하려면 여러 단계를 거쳐야 하므로 매우 흥미로운 일이 될 것입니다.

### 2\. 메서드 서명은 정확히 일치해야 합니다.

이것은 반드시 문제가 되는 것은 아니며, 단지 주의해야 할 사항입니다. 인터셉트하는 메서드의 메서드 인수를 인터셉터 메서드의 매개변수로 간단히 캐스팅할 수 있다고 해도 이는 유효하지 않습니다. 컴파일러는 앞서 설명한 것처럼 무엇을 잘못했는지 설명하는 유용한 오류를 표시합니다.

```bash
error CS9144: Cannot intercept method 'EndpointRouteBuilderExtensions.MapPost
(IEndpointRouteBuilder, string, RequestDelegate)' with interceptor 'Interception.InterceptMapGet(IEndpointRouteBuilder, string, Delegate)' because the signatures do not match.
```

따라서 이 오류를 주의 깊게 읽어보세요! 사용 사례에 따라 [더 많은 메서드를 생성해야 할 수도 있으므로](https://github.com/dotnet/csharplang/issues/7009#issuecomment-1580277384) 주의해야 할 사항입니다.

### 3\. 라이브러리 코드에서 메서드를 가로챌 수 없습니다.

인터셉터는 소스 생성기와 마찬가지로 컴파일 프로세스의 일부로 실행됩니다. 즉, 직접 컴파일하는 코드에서 호출되는 메서드 호출만 인터셉트할 수 있으며, NuGet에서 참조한 라이브러리에서 호출되는 메서드 호출은 인터셉트할 수 없습니다.

구체적인 예를 들어 회사에서 모든 애플리케이션에 추가하는 표준 미니멀 API 엔드포인트 세트가 있다고 가정해 보겠습니다. 작업을 단순화하기 위해 이러한 호출을 캡슐화하는 내부 NuGet 패키지를 만듭니다. 패키지 사용자는 호출하기만 하면 됩니다.

```csharp
app.AddStandardEndpoints();
```

그리고 뒤에서의 `AddStandardEndpoints()` 확장 메서드는 다음과 같이 보입니다.

```csharp
public static class StandardEndpointExtensions
{
    public static IEndpointRouteBuilder AddStandardEndpoints(this IEndpointRouteBuilder app)
    {
        app.MapGet("/healthz", () => "OK");
        app.MapGet("/metrics", () => {});
        // ... etc
    }
}
```

`AddStandardEndpoints()` 메서드는 NuGet 패키지에 정의되어 있으므로 이미 컴파일되어 있습니다. 즉, 미니멀 API 소스 생성기가 해당 `MapGet` 호출을 가로채지 않습니다. 이러한 호출에 대해 AOT를 지원하는 유일한 방법은 라이브러리를 컴파일할 때 미니멀 API 소스 생성기를 활성화한 후 NuGet에 배포하는 것입니다.

이 방법은 여러 면에서 더 간단하지만 "일반적인" 객체 지향 프로그래밍(AOP)을 수행하는 방법으로서 인터셉터의 유용성을 제한합니다.

### 4\. 인터셉터는 완전한 AOP 솔루션이 아닙니다.

이는 3번에서 어느 정도 이어지지만, 메서드 가로체기는는 [PostSharp/Metalama](https://www.postsharp.net/metalama)와 같은 AOP 라이브러리의 일반적인 기능이라는 점을 말씀드리는 것이 중요하다고 생각합니다. 그러나 이러한 라이브러리는 더 많은 AOP 기능과 솔루션을 제공합니다.

인터셉터 설계에 대한 큰 비판 중 하나는 진정한 AOP 솔루션으로 '성장'하는 것을 허용하지 않는다는 것입니다. 개인적으로 이는 타당한 우려라고 생각합니다. 현재 디자인은 매우 "독립형"으로 느껴지는데, 이는 괜찮을 수도 있습니다(특히 제가 절실히 원하는 다른 AOP 접근 방식이 많지 않기 때문에). 그럼에도 불구하고 "단일 독립형 기능"이 아닌 ["AOP 프레임워크" 접근 방식을 채택](https://github.com/dotnet/csharplang/issues/7009#issuecomment-1479711571)하는 것이 컴파일러에 더 강력한 장기적인 솔루션이 될 수 있을 것 같다는 생각이 듭니다. 인터셉터가 아직 실험 단계에 불과하다는 점을 감안하면 아직 변화할 시간이 있습니다!

### 5\. 파일 경로로 인해 생성기 출력 지속에 문제가 발생합니다.

사소한 기능이지만 저는 [소스 생성기 출력을 디스크에 출력](https://andrewlock.net/creating-a-source-generator-part-6-saving-source-generator-output-in-source-control/)하여 소스 제어에 포함할 수 있도록 설정하는 것을 좋아합니다. 이 기능은 부수적인 변경 사항의 영향을 확인할 수 있으므로 생성된 코드의 변경 사항을 포함하여 모든 효과적인 코드 변경 사항을 검토할 수 있으므로 PR의 코드 검토에 특히 유용합니다.

> Datadog에서는 모든 PR에서 실행되는 [GitHub 작업도 있어](https://github.com/DataDog/dd-trace-dotnet/blob/master/.github/workflows/verify_source_generated_changes_are_persisted.yml) 소스 생성기 변경 사항을 올바르게 커밋하고 푸시했는지 확인합니다.

안타깝게도 인터셉터는 소스 코드에 인터셉트 대상의 파일 경로를 리터럴 문자열로 작성해야 하므로 이러한 접근 방식이 깨질 가능성이 높습니다.

```csharp
[InterceptsLocation(@"C:\testapp\Program.cs", 4, 5)]
```

문제는 코드가 어떤 디렉토리에서 체크 아웃되는지, 리눅스, 맥OS 또는 윈도우에서 실행되는지에 따라 파일 경로가 달라진다는 것입니다 🙁 솔직히 이 문제를 어떻게 해결해야 할지 잘 모르겠고, (프로젝트에 두 명 이상이 작업하는 경우) 생성기 코드를 지속하는 것과 근본적으로 호환되지 않는 것 같아서 조금 슬펐습니다.

### 6\. 메서드 교체에 따른 보안 문제

[dotnet/csharplang#7009](https://github.com/dotnet/csharplang/issues/7009) 이슈에서 많은 사람들이 이 문제를 제기했기 때문에 이 문제를 추가했습니다. [예를 들어](https://github.com/dotnet/csharplang/issues/7009#issuecomment-1613192957)

> 어떤 보안 문제가 있나요?
> 
> * 메서드 하이재킹
>     
> * 예를 들어 잠재적으로 악의적인 코드 삽입
>     
> * 중요한 부작용을 무시하는 경우
>     
> * 호출이 완전히 대체되어 발생할 수 있는 사용자 알림 누락
>     

그러나 **인터셉터는 기존 소스 생성기나 NuGet 패키지보다 더 많은 보안 문제를 야기하지 않습니다.**

저는 이 점을 강조하고 싶습니다. 이미 미리보기 6 미니멀 API와 구성 바인더 소스를 통해 인터셉터 없이 메서드 호출을 '하이재킹'하는 것이 완벽하게 가능하다는 것을 보여드렸습니다! 하지만 이를 위해 소스 생성기를 사용할 필요는 없습니다. 예를 들어 기존 확장 메서드를 "재정의"하는 악성 NuGet 패키지로 위의 모든 우려 사항을 달성할 수 있습니다.

따라서 인터셉터에 대한 모든 우려 사항 중에서 보안은 그 중 하나가 되어서는 안 됩니다. 프로젝트에 추가하는 모든 NuGet 패키지를 정말로 신뢰해야 한다는 사실을 강조하는 것일 수도 있습니다.

이미 글이 너무 길어졌기 때문에 [마크 그라벨](https://blog.marcgravell.com/)이 작성한 [Dapper 인터셉터 빌드 도구에 대한 간단한 개념 증명](https://github.com/dotnet/csharplang/issues/7009#issuecomment-1578800679) 결과만 남겨두겠습니다.

> * 빌드 도구를 추가하면 시작 시간이 최대 95ms에서 최대 37ms로 개선되며, 후속 읽기 속도에는 큰 변화가 없습니다(약 270~280ms 유지)
>     
> * 전체 AOT 모드는 시작 시간을 0으로 줄이고 후속 성능을 최대 225ms까지 개선합니다.
>     
> * 물론 DapperAOT를 사용한 전체 AOT 모드는 이제 작동한다는 장점이 있습니다(이전에는 런타임에 실패했습니다)
>     

전반적으로 매우 유망합니다!

## 요약

이 글에서는 실험적인 C#12 기능인 인터셉터에 대해 살펴봤습니다. 메서드 호출을 다른 메서드(일반적으로 소스 생성)로 대체하는 방법을 설명했는데, 예를 들어 기존 구현을 AOT 친화적인 구현으로 대체하는 데 유용할 수 있습니다.

다음으로 인터셉터가 어떻게 작동하는지에 대한 아주 간단한 예시를 `[InterceptsLocation]` 속성을 사용하여 보여드렸습니다. 그런 다음 [이전 게시물](https://dimohy.slogs.dev/exploring-the-dotnet-8-preview-exploring-the-new-minimal-api-source-generator)에서 설명한 접근 방식 대신 인터셉터를 사용하도록 ASP.NET Core 미니멀 API 소스 생성기가 어떻게 업데이트되었는지 보여드렸습니다.

마지막으로 인터셉터에 대한 몇 가지 제한 사항과 문제점에 대해 설명했습니다. 이러한 문제 대부분은 설계 선택과 관련된 것이지만, 제 사용 사례에서 가장 문제가 되는 것은 파일 차이 문제로 인해 소스 생성기 출력을 소스 제어에서 유지할 수 없다는 것입니다. 자주 언급되는 우려 사항 중 하나인 보안은 인터셉터가 타사 NuGet 패키지보다 더 많은 위험을 초래하지 않기 때문에 문제가 되지 않습니다.