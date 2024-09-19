---
title: "NestJS + Swagger 에서 request 에 file 주입하기"
author: yujiniii
date: 2024-09-19 20:00:00 +0900
categories: [Backend, NestJS]
tags: ["backend", "node"]
img_path: "/assets/img/posts/nestjs-file/"
pin: false
---
> 1. NestJS 컨트롤러에서 파일을 body에 담아 업로드하는 방법을 알아본다.
> 2. NestJS + Swagger 에서 파일 업로드를 테스트하는 방법을 알아본다. 
   
## 파일 업로드
파일 업로드를 처리하기 위해 NestJS는 Express용 [multer](https://github.com/expressjs/multer) 미들웨어 패키지를 기반으로 하는 내장 모듈을 제공한다.
multer 는 주로 HTTP POST 요청을 통해 파일을 업로드하는 데 사용되는 `multipart/form-data` 형식으로 게시된 데이터를 처리한다.
  
> [NestJS::file-upload](https://docs.nestjs.com/techniques/file-upload)      
> [mdn::MIME-Type#multipart/form-data](https://developer.mozilla.org/ko/docs/Web/HTTP/Basics_of_HTTP/MIME_types#multipartform-data)

### 구현해보기
  
```bash
$ npm i -D @types/multer
```

```ts
@Post('upload')
@UseInterceptors(FileInterceptor('file'))
uploadFile(@UploadedFile() file: Express.Multer.File) {
  console.log(file);
}
```
FileInterceptor 라는 정의된 인터셉터를 통해 request 로부터 file을 가져오고, @UploadedFile() 데코레이터를 통해 file 을 사용한다.
조금 더 자세히 들여다 보자.

#### FileInterceptor
[nestjs/nest::FileInterceptor](https://github.com/nestjs/nest/blob/093860ad2b3fe8a8a85fd97458fa6bc8041e835a/packages/platform-express/multer/interceptors/file.interceptor.ts#L25)
```ts
export function FileInterceptor(
  fieldName: string,
  localOptions?: MulterOptions,
): Type<NestInterceptor> {
  protected multer: MulterInstance;
  // ...
  async intercept(
    context: ExecutionContext,
    next: CallHandlers,
    ): Promise<Observable<any>> {
      const ctx = context.switchToHttp();
  
      await new Promise<void>((resolve, reject) =>
        this.multer.single(fieldName)(
          ctx.getRequest(),
          ctx.getResponse(),
          (err: any) => {
            if (err) {
              const error = transformException(err);
              return reject(error);
            }
            resolve();
          },
        ),
      );
      return next.handle();
    }
    //...
}

```
intercept() 메서드 부분만 집중해서 보면, 요청에 대해 multer.single() 을 수행하는 것을 확인할 수 있다.
> **`.single(fieldname)` ?**  
> Accept a single file with the name fieldname. The single file will be stored in req.file.  
> [expressjs/multer::readme](https://github.com/expressjs/multer?tab=readme-ov-file#singlefieldname)
  
`.single()` 은 요청에 담긴 파일을 받아서 req.file 에 저장한다.

#### @UploadedFile()  
[nestjs/nest::UploadedFile](https://github.com/nestjs/nest/blob/093860ad2b3fe8a8a85fd97458fa6bc8041e835a/packages/common/decorators/http/route-params.decorator.ts#L240)     
```typescript
// L240
export function UploadedFile(
  fileKey?: string | (Type<PipeTransform> | PipeTransform),
  ...pipes: (Type<PipeTransform> | PipeTransform)[]
): ParameterDecorator {
  return createPipesRouteParamDecorator(RouteParamtypes.FILE)(
    fileKey,
    ...pipes,
  );
}

// L66
const createPipesRouteParamDecorator =
  (paramtype: RouteParamtypes) =>
    (
      data?: any,
      ...pipes: (Type<PipeTransform> | PipeTransform)[]
    ): ParameterDecorator =>
      (target, key, index) => {
        const args =
          Reflect.getMetadata(ROUTE_ARGS_METADATA, target.constructor, key) || {};
        const hasParamData = isNil(data) || isString(data);
        const paramData = hasParamData ? data : undefined;
        const paramPipes = hasParamData ? pipes : [data, ...pipes];

        Reflect.defineMetadata(
          ROUTE_ARGS_METADATA,
          assignMetadata(args, paramtype, index, paramData, ...paramPipes),
          target.constructor,
          key,
        );
      };
```
> * UploadedFile 주석 중 일부
> * Route handler parameter decorator. Extracts the `file` object
> * and populates the decorated parameter with the value of `file`.

`UploadedFile()` 은 데코레이터로, `createPipesRouteParamDecorator(FILE)(..)` 를 그대로 return 한다.  
`createPipesRouteParamDecorator(..)()` 는 request 에서 `paramtype` 에 해당하는 요소를 가져와 메타데이터를 정의하여 사용할 수 있게 만들어준다.   
`paramtype` 는 query, body, header 등 흔히 생각하는 Route handler parameter 항목들을 의미한다.     
    
(`@Query()`, `@Body()` 등 요청 파라미터 데코레이터 구현체를 까보면 `paramtype` 만 다르고 내용은 위와 유사하게 구현되어 있다.)
    
이렇게 구현된 `UploadedFile()` 데코레이터를 통해 변수에 `req.file` 값을 주입할 수 있다.
  
> 틀린 부분이 있을 수도 있으니 느낌만 가져가자..




## Swagger 에서 파일 업로드 테스트하기
### 문제상황
위에서 작성한 코드를 그대로 swagger 에서 사용하면 아래처럼 테스트가 불가능하다.  
![swagger-upload](/swagger.png)

`multipart/form-data` 를 테스트할 수 없기 때문이다.     

### 해결하기 

이를 해결하기 위해서 `ApiConsumes()` 을 사용해 content-type 을 명시하고, 스키마 방식으로 `ApiBody()` 를 정의한다.    
그리고 하나의 데코레이터처럼 사용하기 위해 `applyDecorators()` 를 사용해 잘 말아준다.  
  
```typescript  
import { applyDecorators, UseInterceptors } from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { ApiBody, ApiConsumes } from '@nestjs/swagger';

export function ApiFile(fieldName = 'file') {
  return applyDecorators(
    UseInterceptors(FileInterceptor(fieldName)),
    ApiConsumes('multipart/form-data'),
    ApiBody({
      schema: {
        type: 'object',
        properties: {
          [fieldName]: {
            type: 'string',
            format: 'binary',
          },
          filePath: {
            type: 'string',
            example: 'upload/path/example',
          },
        },
      },
    }),
  );
}

```
   
```typescript
@Post('upload')
// @UseInterceptors(FileInterceptor('file'))
@ApiFile('file')
uploadFile(@UploadedFile() file: Express.Multer.File) {
  console.log(file);
}
```
  
![swagger-upload](/add-api-file.png)
     
짜잔🧙🏻 잘 수정된 것을 확인할 수 있다.  
    
실제로 파일을 넣어 요청을 보내 보았을 때도, console 에서 파일 정보를 확인하였다.  
```bash
# console.log(file)
{
  fieldname: 'file',
  originalname: 'á\x84\x89á\x85³á\x84\x8Fá\x85³á\x84\x85á\x85µá\x86«á\x84\x89á\x85£á\x86º 2024-09-19 á\x84\x8Bá\x85©á\x84\x92á\x85® 9.43.37.png',
  encoding: '7bit',
  mimetype: 'image/png',
  buffer: <Buffer 89 50 4e 47 0d 0a 1a 0a 00 00 00 0d 49 48 44 52 00 00 08 f2 00 00 03 d4 08 06 00 00 00 2d b6 85 57 00 00 0c 3d 69 43 43 50 49 43 43 20 50 72 6f 66 69 ... 96536 more bytes>,
  size: 96586
}
```
  
  
  
###  [고민] ApiConsumes() 를 사용하지 않았을 때 기본 값은 무엇일까?  
작성하면서 궁금증이 생겼다.  
따로 nestjs 나 swagger 문서에 content-type 에 대한 기본값이 명시되어 있지는 않았다.  
오히려 [swagger 3.0 이 되면서 content-type 을 필수로 명시](https://github.com/OAI/OpenAPI-Specification/issues/1031)해야한다는 것 같은 내용만 찾았다..     
  
```typescript
// 1. swagger 정의만 사용
@Post('use-api-body')
@ApiBody({ type: testDto })
test1(body: testDto) {
  console.log(body);
}

// 2. swagger 정의 X, nestjs route parameter decorator 만 사용
@Post('use-route-parameter-decorator')
test2(@Body() body: testDto) {
  console.log(body);
}

// 3. 아무것도 정의 X
@Post('none')
test3(body: testDto) {
  console.log(body);
}
```
   
위와 같은 3가지 case 에서, swagger 는 어떻게 표현될까?  
swagger 를 /api 에 연결했다면, /api-json 으로 자동 생성된 swagger json 을 확인할 수 있다.  
그 결과값은 아래와 같다.  
  

```json
{
  "openapi": "3.0.0",
  "paths": {
    "/use-api-body": {
      "post": {
        "operationId": "AppController_test1",
        "parameters": [],
        "requestBody": {
          "required": true,
          "content": {
            "application/json": {
              "schema": {
                "$ref": "#/components/schemas/testDto" 
              }
            }
          }
        },
        "responses": { "201": { "description": "" }
        }
      }
    },
    "/use-route-parameter-decorator": {
      "post": {
        "operationId": "AppController_test2",
        "parameters": [],
        "requestBody": {
          "required": true,
          "content": {
            "application/json": {
              "schema": {
                "$ref": "#/components/schemas/testDto"
              }
            }
          }
        },
        "responses": { "201": { "description": "" }
        }
      }
    },
    "/none": {
      "post": {
        "operationId": "AppController_test3",
        "parameters": [],
        "responses": { "201": { "description": "" }
        }
      }
    }
  }
}

```

따라서, DTO를 정의하고 body의 타입으로 사용할 때,   
1. `ApiBody()` 를 사용해서 DTO를 지정하는 경우 `application/json` 으로 표현된다.  
2. `@Body()` 를 사용해서 DTO를 지정하는 경우 `application/json` 으로 표현된다.  
3. content 가 명시되지 않는 경우는 body 가 없는 것으로 표현된다.  

**그러면 왜 DTO를 연결했을 때 application/json 으로 사용할까???**  
> [NestJS::Controller](https://docs.nestjs.com/controllers)    
> Using this built-in method, when a request handler returns a JavaScript object or array,  
> it will automatically be serialized to JSON     
  
nestjs 에서 request handler 가 javascript object 또는 array 를 반환하면 자동으로 JSON 으로 직렬화된다고 한다.    
그래서 request parameter 로 DTO 를 사용하면 자동으로 JSON 으로 표현되는 것이다.   
    
따라서, `ApiConsumes()` 를 사용하지 않았을 때는 다른 요소 (@ApiBody(), @Body() 등) 에 따라 content-type 이 결정되는 것으로 보인다.    
(아..마도..?)   


# 마무리
묵혀둔 주제.. ㅎ__ㅎ..  
블로그를 이전하면서 단순히 짜집기식 개념 정리만 하는 것이 아니라, 실험해보고 코드를 뜯어본 것에 대해서만 정리하는 것이 좋겠다고 생각했다.   
블로그를 쓰는데 훨씬 오래걸리고, 버리게 되는 주제도 많긴 하지만 이렇게 정리하니 뭔가 뿌듯한거 같기도 !!!  

얼른 실력을 키워서, 의문과 추측보다는 확신이 가능한 양질의 컨텐츠를 생산하고 싶다  
