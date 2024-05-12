---
title: "custom decorator에 항상 custom pipe 적용하기"
author: yujiniii
date: 2024-05-12 10:00:00 +0900
categories: [Backend, NestJS]
tags: ["backend", "node"]
img_path: "/assets/img/posts/PN0/"
pin: false
---


요구사항
1. custom decorator 를 사용한다. 
2. decorator의 반환값에 custom pipe를 적용하여 validation 한다.
3. controller에서 decorator를 사용할 때는 pipe를 통해 필터링 되고 있음을 숨겨야 한다.
  a. @CustomDecorator(CustomPipe) value: any (x)
  b. @CustomDecorator() value: any (o)

---

> 편의를 위해 CustomPipe는 ParseIntPipe 로 작성하였습니다. 
나중에 적용할 때는 ParseIntPipe 를 CustomPipe 로 변경

--

## 개요

일반적으로 데코레이터에 파이프를 적용한다고 하면 이런 형태가 떠오른다. 
[NestJS::validation](https://docs.nestjs.com/techniques/validation#explicit-conversion)

```ts
// some.controller.ts
@Get(':id')
findOne(
  @Param('id', ParseIntPipe) id: number,
  @Query('sort', ParseBoolPipe) sort: boolean,
) {
  console.log(typeof id === 'number'); // true
  console.log(typeof sort === 'boolean'); // true
  return 'This action returns a user';
}
```

`@Param()` 을 기준으로 찾아보자
IDE에서 따라가보면 `@Param()` 아래와 같은 구조를 갖고 있음을 알 수 있다. 즉 파라미터에 파이프를 받을 수 있게끔 되어 있는 것이다.
```ts
export declare function Param(property: string, ...pipes: (Type<PipeTransform> | PipeTransform)[]): ParameterDecorator;
```


그러면 커스텀 데코레이터에는 파이프를 어떻게 적용할까?
https://docs.nestjs.com/custom-decorators 의 공식 예제인 User를 통해 알아보자
편의를 위해 코드를 조금 변경한다. 

```ts
// user.decorator.ts
export const User = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    // const request = ctx.switchToHttp().getRequest();
    // return request.user;
    return 'i am string'
  },
);

// user.controller.ts
@Get()
async findOne(@User(ParseIntPipe) user: number) { // add pipe
  console.log(user);
  return 'passed!';
}
```

정상적으로 동작하려면, 이 코드는 에러가 나야 한다.
요청해보자.

```bash
$ curl localhost:5000
{"message":"Validation failed (numeric string is expected)","error":"Bad Request","statusCode":400}%  
```

아무 설정 없이도 pipe 주입이 가능한 이유는 무엇일까?
```ts
export declare function createParamDecorator<FactoryData = any, FactoryInput = any, FactoryOutput = any>(factory: CustomParamFactory<FactoryData, FactoryInput, FactoryOutput>, enhancers?: ParamDecoratorEnhancer[]): (...dataOrPipes: (Type<PipeTransform> | PipeTransform | FactoryData)[]) => ParameterDecorator;
```

이렇게 `createParamDecorator` 의 응답타입이 `(...dataOrPipes: (Type<PipeTransform> | PipeTransform | FactoryData)[]) => ParameterDecorator` 이기 때문에, 데코레이터를 생성할 때 별다른 설정을 해주지 않아도 pipe를 파라미터로 받아 처리가 가능하다.

하지만, 요구사항 3을 지키기 위해서는 `createParamDecorator` 에서 pipe가 적용되어 있어야 한다.

## 실험

그렇다면 아래와 같이 수정할 수 있다.

```ts
// user.decorator.ts
export const User = () =>
  createParamDecorator(
    (data: unknown, ctx: ExecutionContext) => {
    // const request = ctx.switchToHttp().getRequest();
    // return request.user;
    return 'i am string'
    },
  )(ParseIntPipe);

```

```bash
> curl localhost:5000
{"message":"Validation failed (numeric string is expected)","error":"Bad Request","statusCode":400}%    
```

우선 어떻게든 적용이 되었음을 확인할 수 있었다. 다만 저 코드에는 심각한 문제가 있다. 타입이 아래처럼 잡혀버렸다. 

```ts
function User(): (target: Object, propertyKey: (string | symbol | undefined), parameterIndex: number) => void
```

첫째, 응답이 void로 왔기 때문에 데코레이터가 제 역할을 하고 있지 못하다. pipe를 통과했더라도 user에는 아무값도 들어오지 않는다.
둘째, 특정 api에서 @User에 추가 pipe를 적용하고 싶은 경우나, 타겟을 직접적으로 선택하고 싶은 경우에 아래 오류와 함께 사용할 수 없다.

`TS2554: Expected 0 arguments, but got 1`

```ts
// 추가 파이프를 적용하는 경우
async findOne(@User(new ValidationPipe({..})) user: any) {..}

// 타겟을 직접적으로 선택하는 경우
async findOne(@User('userId') userId: number) {..}

// 혹은 둘 다
async findOne(@User('userId', new ValidationPipe({..})) userId: number) {..}
```

정상적으로 동작하기 위해서는 기존의 타입을 지켜야 한다. 


## 최종
아직 개발실력이 많이 부족해서, 팀의 시니어 개발자님께서 코드 작성에 도움을 주셨다.  
```ts
// user.decorator.ts
export const User = (...dataOrPipes: (Type<PipeTransform> | PipeTransform)[]) =>
  createParamDecorator(
    (data: unknown, ctx: ExecutionContext) => {
      // const request = ctx.switchToHttp().getRequest();
      // return request.user;
      return 'i am string'
    },
  )(...dataOrPipes, ParseIntPipe);
```

`createParamDecorator의` 타입을 그대로 살리면서, 데코레이터를 바로 실행하도록 코드를 변경하였고,
이때 실행할 때 `@User()` 의 파라미터로 받은 pipe들과 `ParseIntPipe` 를 모두 적용할 수 있었다.

```bash
$ curl localhost:5000
{"message":"Validation failed (numeric string is expected)","error":"Bad Request","statusCode":400}%    
```

pipe가 잘 적용되고 있음을 확인하고, 테스트를 위해 pipe를 통과하도록 `@User()` 리턴값을 변경했다.

```ts
// user.decorator.ts
export const User = (...dataOrPipes: (Type<PipeTransform> | PipeTransform)[]) =>
  createParamDecorator(
    (data: unknown, ctx: ExecutionContext) => {
      // const request = ctx.switchToHttp().getRequest();
      // return request.user;
      return 10;
    },
  )(...dataOrPipes, ParseIntPipe);
```

```bash
$ curl localhost:5000
passed!%       
```

요즘 이런 common 한 코드 작성을 열심히 하려고 하고 있다. 
항상 부족하지만 다음 번엔 조금 더 잘할 수 있겠지!