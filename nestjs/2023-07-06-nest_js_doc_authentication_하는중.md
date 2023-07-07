---
title: "[DOCS] Nest JS 공식문서 (4)"
categories:
    - nest js
tags:
    - nest js

last_modified_at: 2023-07-06T00:20:00
---

[Authentication](https://docs.nestjs.com/security/authentication) 페이지 해석

# 1. Authentication

인증은 대부분의 애플리케이션에서 필수적인 부분입니다. 인증을 처리하기 위한 다양한 접근 방식과 전략이 있습니다. 프로젝트마다 특정 애플리케이션 요구 사항에 따라 접근 방식이 달라집니다. 이 장에서는 다양한 요구 사항에 맞게 조정할 수 있는 인증에 대한 몇 가지 접근 방식을 소개합니다.

요구 사항을 구체화해 봅시다. 이 사용 사례에서 클라이언트는 사용자 이름과 비밀번호로 인증하는 것으로 시작합니다. 인증이 완료되면 서버는 인증을 증명하기 위해 후속 요청 시 인증 헤더에 베어러 토큰으로 전송할 수 있는 JWT를 발급합니다. 또한 유효한 JWT가 포함된 요청만 액세스할 수 있는 보호된 경로를 생성합니다.

첫 번째 요구 사항인 사용자 인증부터 시작하겠습니다. 그런 다음 JWT를 발행하여 이를 확장합니다. 마지막으로 요청에 대해 유효한 JWT를 검사하는 보호된 경로를 생성합니다.

<br>

---

## Creating an authentication module

먼저 `AuthModule` 을 생성하고 그 안에 `AuthService` 와 `AuthController` 를 생성하겠습니다. `AuthService` 를 사용하여 인증 로직을 구현하고 `AuthController` 를 사용하여 인증 엔드포인트를 노출할 것입니다.

```sh
nest g module auth
nest g controller auth
nest g service auth
```

AuthService를 구현하면서 사용자 작업을 UsersService에 캡슐화하는 것이 유용하다는 것을 알게 될 것이므로 이제 해당 모듈과 서비스를 생성해 보겠습니다:

```sh
nest g module users
nest g service users
```

생성된 파일의 기본 내용을 아래와 같이 바꿉니다. 샘플 앱의 경우, `UserService` 는 단순히 하드코딩된 인메모리 사용자 목록과 사용자 이름으로 사용자를 검색하는 찾기 메서드를 유지 관리합니다. 실제 앱에서는 선택한 라이브러리(예: TypeORM, Sequelize, 몽구스 등)를 사용하여 사용자 모델과 지속성 레이어를 빌드할 수 있습니다.

```ts
// users/users.service.ts
import { Injectable } from "@nestjs/common";

// This should be a real class/interface representing a user entity
export type User = any;

@Injectable()
export class UsersService {
    private readonly users = [
        {
            userId: 1,
            username: "john",
            password: "changeme",
        },
        {
            userId: 2,
            username: "maria",
            password: "guess",
        },
    ];

    async findOne(username: string): Promise<User | undefined> {
        return this.users.find((user) => user.username === username);
    }
}
```

`UsersModule` 에서 필요한 유일한 변경 사항은 이 모듈 외부에서 볼 수 있도록 `@Module` 데코레이터의 내보내기 배열에 `UsersService` 를 추가하는 것입니다(곧 `AuthService` 에서 사용하게 될 것입니다).

```ts
// users/users.module.ts

import { Module } from "@nestjs/common";
import { UsersService } from "./users.service";

@Module({
    providers: [UsersService],
    exports: [UsersService],
})
export class UsersModule {}
```

`AuthService`는 사용자를 검색하고 비밀번호를 확인하는 작업을 수행합니다. 이를 위해 `signIn()` 메서드를 생성합니다. 아래 코드에서는 편리한 ES6 스프레드 연산자를 사용하여 사용자 객체에서 비밀번호 속성을 제거한 후 반환합니다. 이는 사용자 객체를 반환할 때 비밀번호나 기타 보안 키와 같은 민감한 필드를 노출하지 않으려는 일반적인 관행입니다.

<br>

---

## Implementing the "Sign in" endpoint

`AuthService` 는 사용자를 검색하고 비밀번호를 확인하는 작업을 수행합니다. 이를 위해 `signIn()` 메서드를 생성합니다. 아래 코드에서는 편리한 ES6 스프레드 연산자를 사용하여 사용자 객체에서 비밀번호 속성을 제거한 후 반환합니다. 이는 사용자 객체를 반환할 때 비밀번호나 기타 보안 키와 같은 민감한 필드를 노출하지 않으려는 일반적인 관행입니다.

```ts
// auth/auth.service.js

import {
    Injectable,
    Dependencies,
    UnauthorizedException,
} from "@nestjs/common";
import { UsersService } from "../users/users.service";

@Injectable()
@Dependencies(UsersService)
export class AuthService {
    constructor(usersService) {
        this.usersService = usersService;
    }

    async signIn(username: string, pass: string) {
        const user = await this.usersService.findOne(username);
        if (user?.password !== pass) {
            throw new UnauthorizedException();
        }
        const { password, ...result } = user;
        // TODO: Generate a JWT and return it here
        // instead of the user object
        return result;
    }
}
```

❗️ 물론 실제 애플리케이션에서는 비밀번호를 일반 텍스트로 저장하지 않습니다. 대신 솔트 처리된 단방향 해시 알고리즘이 포함된 bcrypt와 같은 라이브러리를 사용할 것입니다. 이 접근 방식을 사용하면 해시된 비밀번호만 저장한 다음 저장된 비밀번호를 들어오는 비밀번호의 해시된 버전과 비교하므로 사용자 비밀번호를 일반 텍스트로 저장하거나 노출하지 않습니다. 샘플 앱을 단순하게 유지하기 위해 이러한 절대적인 의무를 위반하고 일반 텍스트를 사용했습니다. 실제 앱에서는 이렇게 하지 마세요!

이제 `AuthModule` 을 업데이트하여 `UsersModule` 을 가져옵니다.

```ts
import { Module } from "@nestjs/common";
import { AuthService } from "./auth.service";
import { AuthController } from "./auth.controller";
import { UsersModule } from "../users/users.module";

@Module({
    imports: [UsersModule],
    providers: [AuthService],
    controllers: [AuthController],
})
export class AuthModule {}
```

이제 AuthController를 열고 signIn() 메서드를 추가해 보겠습니다. 이 메서드는 클라이언트에서 사용자를 인증하기 위해 호출됩니다. 이 메서드는 요청 본문에서 사용자 이름과 비밀번호를 받고, 사용자가 인증되면 JWT 토큰을 반환합니다.

```ts
// auth/auth.controller.js

import { Body, Controller, Post, HttpCode, HttpStatus } from "@nestjs/common";
import { AuthService } from "./auth.service";

@Controller("auth")
export class AuthController {
    constructor(private authService: AuthService) {}

    @HttpCode(HttpStatus.OK)
    @Post("login")
    signIn(@Body() signInDto: Record<string, any>) {
        return this.authService.signIn(signInDto.username, signInDto.password);
    }
}
```

☝🏻 HINT - 이상적으로는 `Record<string, any>` 유형을 사용하는 대신 DTO 클래스를 사용하여 요청 본문의 모양을 정의해야 합니다. 자세한 내용은 [validation](https://docs.nestjs.com/techniques/validation) 챕터를 참조하세요.

<br>

---

## JWT token

이제 인증 시스템의 JWT 부분으로 넘어갈 준비가 되었습니다. 요구 사항을 검토하고 구체화해 보겠습니다.

-   사용자가 사용자 이름/비밀번호로 인증할 수 있도록 허용하여 보호된 API 엔드포인트에 대한 후속 호출에서 사용할 수 있도록 JWT를 반환합니다.

-   bearer token 으로 유효한 JWT의 존재를 기반으로 보호되는 API 경로를 생성합니다.

JWT 요구 사항을 지원하기 위해 하나의 추가 패키지를 설치해야 합니다:

```sh
$ npm install --save @nestjs/jwt
```

☝🏻 HINT - `@nestjs/jwt` 패키지(자세한 내용은 [여기](https://github.com/nestjs/jwt)를 참조하세요)는 JWT 조작에 도움이 되는 유틸리티 패키지입니다. 여기에는 JWT 토큰 생성 및 확인이 포함됩니다.

서비스를 깔끔하게 모듈화하기 위해 `authService`에서 JWT 생성을 처리합니다. `auth` 폴더에서 `auth.service.ts` 파일을 열고 `JwtService` 를 삽입한 다음 `signIn` 메서드를 업데이트하여 아래와 같이 JWT 토큰을 생성합니다.

```ts
// auth/auth.service.ts

import {
    Injectable,
    Dependencies,
    UnauthorizedException,
} from "@nestjs/common";
import { UsersService } from "../users/users.service";
import { JwtService } from "@nestjs/jwt";

@Dependencies(UsersService, JwtService)
@Injectable()
export class AuthService {
    constructor(usersService, jwtService) {
        this.usersService = UsersService;
        this.jwtService = JwtService;
    }

    async signIn(username, pass) {
        const user = await this.usersService.findOne(username);
        if (user?.password !== pass) {
            throw new UnauthorizedException();
        }
        const payload = { username: user.username, sub: user.userId };
        return {
            access_token: await this.jwtService.signAsync(payload),
        };
    }
}
```

사용자 객체 프로퍼티의 하위 집합에서 JWT를 생성하기 위해 `signAsync()` 함수를 제공하는 `@nestjs/jwt` 라이브러리를 사용하고 있으며, 이 함수는 단일 `access_token` 프로퍼티가 있는 간단한 객체로 반환합니다. 참고: JWT 표준과 일관성을 유지하기 위해 `sub` 라는 속성 이름을 선택하여 `userId` 값을 보유합니다. 인증 서비스에 `JwtService` 공급자를 삽입하는 것을 잊지 마세요.

이제 새 종속성을 가져오고 `JwtModule` 을 구성하기 위해 `AuthModule` 을 업데이트해야 합니다.

먼저 인증 폴더에 `constants.ts`를 생성하고 다음 코드를 추가합니다.

```ts
// auth/constants.js
export const jwtConstants = {
    secret: "DO NOT USE THIS VALUE. INSTEAD, CREATE A COMPLEX SECRET AND KEEP IT SAFE OUTSIDE OF THE SOURCE CODE.",
};
```

이 키를 사용하여 JWT 서명 및 확인 단계 간에 키를 공유할 것입니다.

❗️ WARNING - 이 키를 공개적으로 노출하지 마세요. 여기서는 코드가 수행하는 작업을 명확히 하기 위해 공개했지만, 프로덕션 시스템에서는 시크릿 볼트, 환경 변수 또는 구성 서비스 등의 적절한 방법을 사용하여 이 키를 보호해야 합니다.

이제 인증 폴더에서 `auth.module.ts` 를 열고 다음과 같이 업데이트합니다:

```ts
// auth/auth.module.ts

import { Module } from "@nestjs/common";
import { AuthService } from "./auth.service";
import { UsersModule } from "../users/users.module";
import { JwtModule } from "@nestjs/jwt";
import { AuthController } from "./auth.controller";
import { jwtConstants } from "./constants";

@Module({
    imports: [
        UsersModule,
        JwtModule.register({
            global: true,
            secret: jwtConstants.secret,
            signOptions: { expiresIn: "60s" },
        }),
    ],
    providers: [AuthService],
    controllers: [AuthController],
    exports: [AuthService],
})
export class AuthModule {}
```

☝🏻 HINT - 작업을 더 쉽게 하기 위해 `JwtModule` 을 글로벌로 등록하고 있습니다. 즉, 애플리케이션의 다른 곳에서 `JwtModule` 을 임포트할 필요가 없습니다.

구성 객체를 전달하여 `register()` 를 사용하여 `JwtModule` 을 구성합니다. Nest `JwtModule` 에 대한 자세한 내용은 [여기](https://github.com/auth0/node-jsonwebtoken#usage)를, 사용 가능한 구성 옵션에 대한 자세한 내용은 여기를 참조하세요.

계속해서 cURL을 사용하여 경로를 다시 테스트해 보겠습니다. `UsersService` 에 하드코딩된 모든 `user` 객체를 사용하여 테스트할 수 있습니다.

<br>

---

## Implementing the authentication guard

이제 마지막 요구 사항인 요청에 유효한 JWT가 있어야 엔드포인트를 보호하는 문제를 해결할 수 있습니다. 경로를 보호하는 데 사용할 수 있는 `AuthGuard` 를 생성하여 이를 수행합니다.

```ts
// auth/auth.guard.ts

import {
    CanActivate,
    ExecutionContext,
    Injectable,
    UnauthorizedException,
} from "@nestjs/common";
import { JwtService } from "@nestjs/jwt";
import { jwtConstants } from "./constants";
import { Request } from "express";

@Injectable()
export class AuthGuard implements CanActivate {
    constructor(private jwtService: JwtService) {}

    async canActivate(context: ExecutionContext): Promise<boolean> {
        const request = context.switchToHttp().getRequest();
        const token = this.extractTokenFromHeader(request);
        if (!token) {
            throw new UnauthorizedException();
        }
        try {
            const payload = await this.jwtService.verifyAsync(token, {
                secret: jwtConstants.secret,
            });
            // 💡 We're assigning the payload to the request object here
            // so that we can access it in our route handlers
            request["user"] = payload;
        } catch {
            throw new UnauthorizedException();
        }
        return true;
    }

    private extractTokenFromHeader(request: Request): string | undefined {
        const [type, token] = request.headers.authorization?.split(" ") ?? [];
        return type === "Bearer" ? token : undefined;
    }
}
```

이제 보호 경로를 구현하고 `AuthGard` 를 등록하여 보호할 수 있습니다.

`auth.controller.ts` 파일을 열고 아래와 같이 업데이트합니다.

```ts
import {
    Body,
    Controller,
    Get,
    HttpCode,
    HttpStatus,
    Post,
    Request,
    UseGuards,
} from "@nestjs/common";
import { AuthGuard } from "./auth.guard";
import { AuthService } from "./auth.service";

@Controller("auth")
export class AuthController {
    constructor(private authService: AuthService) {}

    @HttpCode(HttpStatus.OK)
    @Post("login")
    signIn(@Body() signInDto: Record<string, any>) {
        return this.authService.signIn(signInDto.username, signInDto.password);
    }

    @UseGuards(AuthGuard)
    @Get("profile")
    getProfile(@Request() req) {
        return req.user;
    }
}
```

방금 생성한 `AuthGuard` 를 `GET /profile` 경로에 적용하여 보호할 수 있도록 합니다.

앱이 실행 중인지 확인하고 `cURL`을 사용하여 경로를 테스트합니다.

```sh
$ # GET /profile
$ curl http://localhost:3000/auth/profile
{"statusCode":401,"message":"Unauthorized"}

$ # POST /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm..."}

$ # GET /profile using access_token returned from previous step as bearer code
$ curl http://localhost:3000/auth/profile -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm..."
{"sub":1,"username":"john","iat":...,"exp":...}
```

`AuthModule` 에서 JWT의 만료 시간을 `60초` 로 구성했습니다. 이는 너무 짧은 만료 시간이며 토큰 만료 및 새로 고침에 대한 세부 사항을 다루는 것은 이 문서의 범위를 벗어납니다. 하지만 JWT의 중요한 특성을 보여주기 위해 이 방법을 선택했습니다. 인증 후 60초를 기다렸다가 `GET /auth/profile` 요청을 시도하면 `401 Unauthorized` 없음 응답을 받게 됩니다. 이는 `@nestjs/jwt` 가 자동으로 JWT의 만료 시간을 확인하므로 애플리케이션에서 직접 확인해야 하는 수고를 덜어주기 때문입니다.

<br>

---

## Enable authentication globally

대부분의 엔드포인트를 기본적으로 보호해야 하는 경우, 인증 가드를 [global guard](https://docs.nestjs.com/guards#binding-guards) 로 등록하고 각 컨트롤러 위에 `@UseGuards()` 데코레이터를 사용하는 대신 어떤 경로를 공개해야 하는지 플래그를 지정하면 됩니다.

먼저, 다음 구성을 사용하여 `AuthGuard` 를 전역 가드로 등록합니다(예: `AuthModule` 등 모든 모듈에서).

```ts
providers: [
  {
    provide: APP_GUARD,
    useClass: AuthGuard,
  },
],
```

이 설정이 완료되면 Nest는 모든 엔드포인트에 `AuthGuard` 를 자동으로 바인딩합니다.

이제 경로를 공개로 선언하는 메커니즘을 제공해야 합니다. 이를 위해 `SetMetadata` 데코레이터 팩토리 함수를 사용하여 사용자 정의 데코레이터를 만들 수 있습니다.

<br>

---

## Passport integration

[Passport](https://github.com/jaredhanson/passport)는 커뮤니티에서 잘 알려져 있고 많은 프로덕션 애플리케이션에서 성공적으로 사용되는 가장 인기 있는 node.js 인증 라이브러리입니다. 이 라이브러리를 `Nest` 애플리케이션과 통합하는 방법은 `@nestjs/passport` 모듈을 사용하면 간단합니다.

Passport와 NestJS를 통합하는 방법을 알아보려면 이 [챕터](https://docs.nestjs.com/recipes/passport)를 확인하세요.

<br>

---

## Example

이 장의 전체 코드 버전은 [여기](https://github.com/nestjs/nest/tree/master/sample/19-auth-jwt)에서 확인할 수 있습니다.
